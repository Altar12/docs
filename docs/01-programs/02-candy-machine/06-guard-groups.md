---
description: "Explains how to configure multiple groups of guards."
---

import { Accordion, AccordionItem } from '/src/accordion.jsx';

# Guard Groups

## Introduction

On [the previous page](/programs/candy-machine/candy-guards), we introduced guards and used them to define the access control of our Candy Machines. We’ve seen that using guards, we can for instance add payments of 1 SOL per mint and ensure the mint start after a certain date. But what if we also wanted to charge 2 SOL after a second date? What if we wanted to allow certain token holders to mint for free or at a discounted price?

What if we could define multiple sets of guards that each have their own requirements? For that reason, we’ve created **Guard Groups**!

## How Do Groups Work?

Remember [how we can set up guards on any Candy Machine](/programs/candy-machine/candy-guards#creating-a-candy-machine-with-guards) by simply providing the settings of the guards we want to enable? Well, Guard Groups work the same way, except you must also give them a unique **Label** to identify them.

Therefore, each Guard Group has the following attributes:

- **Label**: A unique text identifier. This cannot be longer than 6 characters.
- **Guards**: The settings for all activated guards within that group. This works just like setting up guards directly on the Candy Machine.

Let’s take a quick example. Say we wanted to charge 1 SOL from 4 pm to 5 pm and then 2 SOL from 5 pm until the Candy Machine is exhausted. All of that whilst making sure we are protected against bots via the Bot Tax guard. Here’s how we could set up our guards:

- Group 1:
  - **Label**: “early”
  - **Guards**:
    - Sol Payment: 1 SOL
    - Start Date: 4 pm (ignoring the actual date here for the sake of simplicity)
    - End Date: 5 pm
    - Bot Tax: 0.001 SOL
- Group 2:
  - **Label**: “late”
  - **Guards**:
    - Sol Payment: 2 SOL
    - Start Date: 5 pm
    - Bot Tax: 0.001 SOL

And just like that, we’ve created a customized 2-tier minting process!

Now, whenever someone tries to mint from our Candy Machine, **they will have to explicitly tell us which group they are minting from**. Asking for the group label when minting is important because:

- It ensures buyers do not experience unexpected minting behaviour. Say we tried to mint for 1 SOL at the very end of the first group’s end date but, by the time the transaction executes, we’re now past that date. If we didn’t ask for the group label, the transaction would succeed and we would be charged 2 SOL even though we expected to only be charged 1 SOL.
- It makes it possible to support parallel groups. We’ll talk more about this later on this page.

![CandyMachinesV3-GuardGroups1.png](/assets/candy-machine-v3/CandyMachinesV3-GuardGroups1.png#radius)

Now let’s see how we can create and update groups using our SDKs.

<Accordion>
<AccordionItem title="JS SDK" open={true}>
<div className="accordion-item-padding">

To create Candy Machines with guard groups, simply provide the `groups` array to the `create` operation. Each item of this array must contain a `label` and a `guards` object containing the settings of all guards we wish to activate in that group.
Here’s how we’d implement the above example using the JS SDK.

```tsx
import { sol, toDateTime } from "@metaplex-foundation/js";

const { candyMachine } = await metaplex.candyMachines().create({
  // ...
  groups: [
    {
      label: "early",
      guards: {
        solPayment: { amount: sol(1), destination: treasury },
        startDate: { date: toDateTime("2022-10-18T16:00:00Z") },
        endDate: { date: toDateTime("2022-10-18T17:00:00Z") },
        botTax: { lamports: sol(0.001), lastInstruction: true },
      },
    },
    {
      label: "late",
      guards: {
        solPayment: { amount: sol(2), destination: treasury },
        startDate: { date: toDateTime("2022-10-18T17:00:00Z") },
        botTax: { lamports: sol(0.001), lastInstruction: true },
      },
    },
  ],
});
```

To update groups, simply provide that same `groups` attribute to the `update` operation.
Please note that the entire `groups` array will be updated meaning **it will override all existing groups**!
Therefore, make sure to provide the settings for all your groups, even if their settings are not changing.
Here’s an example, changing the SOL payment guard for the “late” group to 3 SOL instead of 2 SOL.

```tsx
import { sol, toDateTime } from "@metaplex-foundation/js";

await metaplex.candyMachines().update({
  candyMachine,
  groups: [
    {
      label: "early",
      guards: {
        solPayment: { amount: sol(1), destination: treasury },
        startDate: { date: toDateTime("2022-10-18T16:00:00Z") },
        endDate: { date: toDateTime("2022-10-18T17:00:00Z") },
        botTax: { lamports: sol(0.001), lastInstruction: true },
      },
    },
    {
      label: "late",
      guards: {
        solPayment: { amount: sol(3), destination: treasury },
        startDate: { date: toDateTime("2022-10-18T17:00:00Z") },
        botTax: { lamports: sol(0.001), lastInstruction: true },
      },
    },
  ],
});
```

API References:

- Create: [Operation](https://metaplex-foundation.github.io/js/classes/js.CandyMachineClient.html#create), [Input](https://metaplex-foundation.github.io/js/types/js.CreateCandyMachineInput.html), [Output](https://metaplex-foundation.github.io/js/types/js.CreateCandyMachineOutput.html), [Transaction Builder](https://metaplex-foundation.github.io/js/classes/js.CandyMachineBuildersClient.html#create).
- Update: [Operation](https://metaplex-foundation.github.io/js/classes/js.CandyMachineClient.html#update), [Input](https://metaplex-foundation.github.io/js/types/js.UpdateCandyMachineInput.html), [Output](https://metaplex-foundation.github.io/js/types/js.UpdateCandyMachineOutput.html), [Transaction Builder](https://metaplex-foundation.github.io/js/classes/js.CandyMachineBuildersClient.html#update).
- Guard Settings: [Default Candy Guard Settings](https://metaplex-foundation.github.io/js/types/js.DefaultCandyGuardSettings.html).

</div>
</AccordionItem>
</Accordion>

## Default Guards

Notice how, in the example above, we had to provide the same **Bot Tax** guard to both groups. This can be simplified by leveraging the global **Guards** that are set on a Candy Machine.

When using Guard Groups, the global Guards of a Candy Machine — as explained on [the previous page](/programs/candy-machine/candy-guards) — **act as default guards**! That means groups will default to using the same guard settings as the global guards unless they are overriding them by explicitly enabling them in the group.

Here’s a quick recap:

- If a guard is enabled on the default guards but not on the group’s guards, the group uses the guard **as defined in the default guards**.
- If a guard is enabled on the default guards _and_ on the group’s guards, the group uses the guard **as defined in the group’s guards**.
- If a guard is not enabled on the default guards or the group’s guards, the group does not use this guard.

To illustrate that, let’s take our example from the previous section and move the **Bot Tax** guard to the default guards.

- Default Guards:
  - Bot Tax: 0.001 SOL
- Group 1:
  - **Label**: “early”
  - **Guards**:
    - Sol Payment: 1 SOL
    - Start Date: 4 pm
    - End Date: 5 pm
- Group 2:
  - **Label**: “late”
  - **Guards**:
    - Sol Payment: 2 SOL
    - Start Date: 5 pm

As you can see, default guards are useful to avoid repetition within your groups.

![CandyMachinesV3-GuardGroups2.png](/assets/candy-machine-v3/CandyMachinesV3-GuardGroups2.png#radius)

Note that, even when using default guards, a group must be provided when minting. That means, when using guard groups, **it is not possible to mint using the default guards only**.

<Accordion>
<AccordionItem title="JS SDK" open={true}>
<div className="accordion-item-padding">

To use default guards in the JS SDK, simply use the `guards` attribute in conjunction with the `groups` array when creating or updating a Candy Machine.
For instance, here’s how you’d create a Candy Machine using the guard settings described above.

```tsx
import { sol, toDateTime } from "@metaplex-foundation/js";

const { candyMachine } = await metaplex.candyMachines().create({
  // ...
  guards: {
    botTax: { lamports: sol(0.001), lastInstruction: true },
  },
  groups: [
    {
      label: "early",
      guards: {
        solPayment: { amount: sol(1), destination: treasury },
        startDate: { date: toDateTime("2022-10-18T16:00:00Z") },
        endDate: { date: toDateTime("2022-10-18T17:00:00Z") },
      },
    },
    {
      label: "late",
      guards: {
        solPayment: { amount: sol(2), destination: treasury },
        startDate: { date: toDateTime("2022-10-18T17:00:00Z") },
      },
    },
  ],
});
```

API References: [Operation](https://metaplex-foundation.github.io/js/classes/js.CandyMachineClient.html#create), [Input](https://metaplex-foundation.github.io/js/types/js.CreateCandyMachineInput.html), [Output](https://metaplex-foundation.github.io/js/types/js.CreateCandyMachineOutput.html), [Transaction Builder](https://metaplex-foundation.github.io/js/classes/js.CandyMachineBuildersClient.html#create), [Default Candy Guard Settings](https://metaplex-foundation.github.io/js/types/js.DefaultCandyGuardSettings.html).

</div>
</AccordionItem>
</Accordion>

## Parallel Groups

One really interesting benefit of requiring the group label when minting is the ability to have **more than one valid group at a given time**. This removes any ambiguity for the program and allows the buyer to select which group they would like to try to mint from.

Let’s illustrate that with a new example. Say we have an NFT collection called “Innocent Bird” and we want to offer a discounted price of 1 SOL to anyone holding an “Innocent Bird” NFT and charge anyone else 2 SOL. We want both of these groups to be able to start minting at the same time — say 4 pm — and we also want to be protected against bots for both groups. Here’s how we could configure our guards:

- Default Guards:
  - Start Date: 4 pm
  - Bot Tax: 0.001 SOL
- Group 1:
  - **Label**: “nft”
  - **Guards**:
    - Sol Payment: 1 SOL
    - NFT Gate: “Innocent Bird” Collection
- Group 2:
  - **Label**: “public”
  - **Guards**:
    - Sol Payment: 2 SOL

As you can see, with these guard settings, it is possible for both groups to mint at the same time. It is even possible for an NFT holder to pay the full 2 SOL should they decide to mint from the “public” group. However, it is in their best interest to select the “nft” group if they can.

<Accordion>
<AccordionItem title="JS SDK" open={true}>
<div className="accordion-item-padding">

Here’s how you’d create a Candy Machine using the guard settings described above via the JS SDK.

```tsx
import { sol, toDateTime } from "@metaplex-foundation/js";

const { candyMachine } = await metaplex.candyMachines().create({
  // ...
  guards: {
    botTax: { lamports: sol(0.001), lastInstruction: true },
    startDate: { date: toDateTime("2022-10-18T16:00:00Z") },
  },
  groups: [
    {
      label: "early",
      guards: {
        solPayment: { amount: sol(1), destination: treasury },
        nftGate: { requiredCollection: innocentBirdCollectionNft.address },
      },
    },
    {
      label: "late",
      guards: {
        solPayment: { amount: sol(2), destination: treasury },
      },
    },
  ],
});
```

API References: [Operation](https://metaplex-foundation.github.io/js/classes/js.CandyMachineClient.html#create), [Input](https://metaplex-foundation.github.io/js/types/js.CreateCandyMachineInput.html), [Output](https://metaplex-foundation.github.io/js/types/js.CreateCandyMachineOutput.html), [Transaction Builder](https://metaplex-foundation.github.io/js/classes/js.CandyMachineBuildersClient.html#create), [Default Candy Guard Settings](https://metaplex-foundation.github.io/js/types/js.DefaultCandyGuardSettings.html).

</div>
</AccordionItem>
</Accordion>

## Conclusion

Guard groups bring a whole new dimension to our Candy Machines by allowing us to define sequential and/or parallel minting workflows tailored to our needs.

On [the next page](/programs/candy-machine/special-guard-instructions), we’ll see yet another exciting feature about guards: Guard instructions!
