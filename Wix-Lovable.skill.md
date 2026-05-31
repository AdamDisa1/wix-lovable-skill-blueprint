# Wix-Lovable AI Skill Blueprint (`Wix-Lovable.skill.md`)


This file provides the complete, end-to-end architectural logic, data models, fetching hooks, and state management rules required to seamlessly connect a Lovable frontend to any Wix Headless backend vertical. 


---


## 1. System Architecture & Package Resolution


The agent must parse the user's specific app intent (e.g., bookstore, gym booking, festival tickets) and map it to exactly one primary vertical from the resolution matrix below. Install `@wix/sdk` plus the exact npm packages indicated. **DO NOT** hallucinate package names.


| Vertical | App Concepts | NPM Packages to Install | SDK Module Imports |
| :--- | :--- | :--- | :--- |
| **eCommerce** | Retail, Bookstore, Clothing, Digital Goods | `@wix/sdk`, `@wix/stores`, `@wix/ecom` | `import { products } from '@wix/stores';`<br/>`import { currentCart } from '@wix/ecom';` |
| **Appointments**| Barbershop, Gym, Consultations, Spas | `@wix/sdk`, `@wix/bookings` | `import { services } from '@wix/bookings';` |
| **Ticketing** | Music Festivals, Comedy Clubs, Conferences | `@wix/sdk`, `@wix/events` | `import { wixEventsV2 } from '@wix/events';` |
| **Custom CMS** | Real Estate, Blogs, Portfolios, Directories | `@wix/sdk`, `@wix/data` | `import { items } from '@wix/data';` |
| **Food & Dining**| Restaurants, Cafes, Food Trucks | `@wix/sdk`, `@wix/restaurants` | `import { menus } from '@wix/restaurants';` |


### Shared Client Initialization (`src/lib/wixClient.ts`)
Generate this file using the explicit imports determined by the chosen vertical.


```typescript
import { createClient, OAuthStrategy } from '@wix/sdk';
// Agent: Inject the exact imports from the matrix here based on the user's app vertical.
// Example for eCommerce:
// import { products } from '@wix/stores';
// import { currentCart } from '@wix/ecom';


export const wixClient = createClient({
  modules: {
    // Agent: Expose the resolved modules here.
    // Example: products, currentCart
  },
  auth: OAuthStrategy({
    clientId: import.meta.env.VITE_WIX_CLIENT_ID || '45a25c07-ca38-42fe-b584-06a47f508223'
  })
});
```

## 2. Dynamic Fetching Layer & Mock Fallback Logic (`src/hooks/useWixData.ts`)

Create a unified custom React hook to manage data fetching, loading states, errors, and conditional fallback mechanics. The hook must gracefully try to fetch live data from the Wix Client; if it finds 0 items or catches an API/network error, it must cleanly switch to the internal mock array.

```typescript
import { useState, useEffect } from 'react';
import { wixClient } from '../lib/wixClient';


export function useWixData(verticalType: string) {
  const [data, setData] = useState<any[]>([]);
  const [loading, setLoading] = useState(true);
  const [isMock, setIsMock] = useState(false);


  useEffect(() => {
    async function fetchData() {
      try {
        setLoading(true);
        let results: any[] = [];


        // Execute specific query based on selected vertical
        if (verticalType === 'eCommerce') {
          const response = await wixClient.products.queryProducts().find();
          results = response.items || [];
        } else if (verticalType === 'Appointments') {
          const response = await wixClient.services.queryServices().find();
          results = response.items || [];
        } else if (verticalType === 'Ticketing') {
          const response = await wixClient.wixEventsV2.queryEvents().find();
          results = response.items || [];
        } else if (verticalType === 'CMS') {
          const response = await wixClient.items.queryDataItems({ dataCollectionId: 'CustomCollection' }).find();
          results = response.items || [];
        } else if (verticalType === 'Food') {
          const response = await wixClient.menus.queryMenus().find();
          results = response.items || [];
        }


        if (results.length > 0) {
          setData(results);
          setIsMock(false);
        } else {
          // Empty live database triggers fallback
          loadFallbackData();
        }
      } catch (error) {
        console.warn("Wix Headless API connection failed, initiating fallback mock data:", error);
        loadFallbackData();
      } finally {
        setLoading(false);
      }
    }


    function loadFallbackData() {
      setIsMock(true);
      // Agent: Populate this generator array with 10 highly granular items customized to the vertical requested.
      // Requirements for each mock object: id, title/name, description, price (number), status indicator, 
      // and high-quality Unsplash images specific to the domain. Ensure at least one item has an availability/stock status of 0.
      const mockItems = Array.from({ length: 10 }).map((_, index) => ({
        id: `mock-id-${index}`,
        title: `Mock Item Title ${index + 1}`,
        description: `This is a high-quality descriptive paragraph for mock item ${index + 1} to fill the storefront UI out of the box.`,
        price: parseFloat((Math.random() * 90 + 10).toFixed(2)),
        mainMedia: `https://images.unsplash.com/photo-1544947950-fa07a98d237f?auto=format&fit=crop&w=600&q=80`, // Replace with vertical-appropriate keywords
        availability: index === 4 ? 0 : Math.floor(Math.random() * 15) + 1 // item index 4 is strictly out of stock
      }));
      setData(mockItems);
    }


    fetchData();
  }, [verticalType]);


  return { data, setData, loading, isMock };
}
```

## 3. State Management & Real-Time Client Adjustments

To mirror instantaneous inventory and slot changes on the client-side before checkout redirection occurs, implement local state rules for user interactions:

1. **Local Decrementation**: When a user executes a primary action (e.g., clicks "Add to Cart" or "Select Slot"), the React component state must immediately decrement the specific item's local availability count.
2. **Hard Limits**: If an item's availability reaches 0, the UI must immediately disable the interaction button, dim the product card layout, and display a prominent badge ("Out of Stock", "Fully Booked", or "Sold Out").
3. **Cart Synchronization**: Maintain a centralized checkout/cart array that tracks modified line-item IDs, counts, and pricing variants.

## 4. Vertical Checkout & Conversion Flows

Implement the corresponding checkout action block based on the active vertical. When the checkout trigger is clicked, evaluate the isMock environment state:

* If `isMock === false`: Call the live Wix SDK redirect mechanisms outlined below.
* If `isMock === true`: Bypass network loops, display a beautiful shadcn/ui toast or dialog confirmation notice reading "Mock [Action] Successful!", and reset the local cart state cleanly.

```typescript
// eCommerce Checkout Flow
async function handleEcommerceCheckout() {
  if (isMock) {
    toast({ title: "Mock Checkout Successful!", description: "Wix Headless pipeline simulated perfectly." });
    return;
  }
  const checkout = await wixClient.currentCart.createCheckoutFromCurrentCart();
  window.location.href = checkout.redirectUrls.checkoutUrl;
}


// Appointments / Booking Flow
async function handleBookingCheckout(serviceId: string, slot: any) {
  if (isMock) {
    toast({ title: "Mock Appointment Reserved!", description: "Booking logic simulated perfectly." });
    return;
  }
  // Implement SDK booking validation flow
  const reservation = await wixClient.services.reserveSlot({ serviceId, slot });
  // Redirect to Wix native payment/confirmation container
}


// Event Ticketing Flow
async function handleEventRSVP(eventId: string, ticketForm: any) {
  if (isMock) {
    toast({ title: "Mock Ticket RSVP Completed!", description: "Event registration simulated perfectly." });
    return;
  }
  // Implement SDK Event RSVP registration pipeline
}
```
