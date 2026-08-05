# eBay Case Study: Auction Ending Surge

## Challenge
Millions of auctions end simultaneously
Peak: 1M+ bid/second in final minute
Normal: 10K bid/second

## Solution
1. **Predictive Scaling:** Add 500+ servers at T-1 hour
2. **Load Balancing:** Distributed by auction ID
3. **Proxy Bidding:** Reduce last-second manual bids
4. **Queue Management:** Hold bids if overload, process in order
5. **Database:** Real-time consistency across shards

## Result
Successfully handles 100x normal load
No auction lost due to system failure
99.99% uptime maintained

