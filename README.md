# SinemTokgoz-BootCamp-FinalProject

use ink_lang as ink;

#[ink::contract]
mod auction {
    use ink_prelude::string::String;
    use ink_prelude::vec::Vec;
    use ink_storage::collections::{HashMap as StorageHashMap, Vec as StorageVec};

    #[ink(storage)]
    pub struct Auction {
        owner: AccountId,
        items: StorageVec<Item>,
        bids: StorageHashMap<ItemId, StorageVec<Bid>>,
    }

    #[derive(Debug, Clone, PartialEq, Eq, scale::Encode, scale::Decode)]
    #[cfg_attr(feature = "std", derive(scale_info::TypeInfo))]
    pub struct Item {
        owner: AccountId,
        name: String,
        price: Balance,
        active: bool,
    }

    #[derive(Debug, Clone, PartialEq, Eq, scale::Encode, scale::Decode)]
    #[cfg_attr(feature = "std", derive(scale_info::TypeInfo))]
    pub struct Bid {
        bidder: AccountId,
        amount: Balance,
    }

    type AccountId = [u8; 32];
    type ItemId = u64;

    #[ink(event)]
    pub struct ItemListed {
        #[ink(topic)]
        item_id: ItemId,
        name: String,
        price: Balance,
    }

    #[ink(event)]
    pub struct BidPlaced {
        #[ink(topic)]
        item_id: ItemId,
        bidder: AccountId,
        amount: Balance,
    }

    #[ink(event)]
    pub struct ItemUpdated {
        #[ink(topic)]
        item_id: ItemId,
        name: String,
        price: Balance,
    }

    #[ink(event)]
    pub struct ItemStopped {
        #[ink(topic)]
        item_id: ItemId,
        new_owner: AccountId,
    }

    impl Auction {
        #[ink(constructor)]
        pub fn new() -> Self {
            let caller = Self::env().caller();
            Self {
                owner: caller,
                items: StorageVec::new(),
                bids: StorageHashMap::new(),
            }
        }

        pub fn list_item(&mut self, name: String, price: Balance) {
            let item_id = self.items.len() as ItemId;
            let owner = self.env().caller();
            let item = Item {
                owner,
                name,
                price,
                active: true,
            };
            self.items.push(item);

            self.env().emit_event(ItemListed {
                item_id,
                name,
                price,
            });
        }

        pub fn place_bid(&mut self, item_id: ItemId, amount: Balance) {
            let caller = self.env().caller();
            let item = &mut self.items[item_id as usize];
            assert!(item.active, "Item is not active");
            assert_ne!(caller, item.owner, "You cannot bid on your own item");
            assert!(amount > item.price, "Bid must be higher than the current price");

            let bid = Bid {
                bidder: caller,
                amount,
            };
            self.bids.entry(item_id).or_insert(Vec::new()).push(bid);

            self.env().emit_event(BidPlaced {
                item_id,
                bidder: caller,
                amount,
            });
        }

        pub fn update_item(&mut self, item_id: ItemId, name: String, price: Balance) {
            let caller = self.env().caller();
            let item = &mut self.items[item_id as usize];
            assert_eq!(caller, item.owner, "Only the owner can update the item");

            item.name = name;
            item.price = price;

            self.env().emit_event(ItemUpdated {
                item_id,
                name,
                price,
            });
        }

        pub fn stop_item(&mut self, item_id: ItemId) {
            let caller = self.env().caller();
            let item = &mut self.items[item_id as usize];
            assert_eq!(caller, item.owner, "Only the owner can stop the item");

            let mut highest_bidder = item.owner;
            let mut highest_bid = item.price;

            if let Some(bids) = self.bids.get(&item_id) {
                for bid in bids.iter() {
                    if bid.amount > highest_bid {
                        highest_bidder = bid.bidder;
                        highest_bid = bid.amount;
                    }
                }
            }

            item.active = false;
            item.owner = highest_bidder;

            self.env().emit_event(ItemStopped {
                item_id,
                new_owner: highest_bidder,
            });
        }

        pub fn get_item(&self, item_id: ItemId) -> Option<&Item> {
            self.items.get(item_id as usize)
        }

        pub fn get_items_count(&self) -> u64 {
            self.items.len() as u64
        }
    }
}
