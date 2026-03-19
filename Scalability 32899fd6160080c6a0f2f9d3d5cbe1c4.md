# Scalability

![Screenshot 2026-03-19 at 7.55.50 PM.png](Scalability/Screenshot_2026-03-19_at_7.55.50_PM.png)

![Screenshot 2026-03-19 at 7.56.40 PM.png](Scalability/Screenshot_2026-03-19_at_7.56.40_PM.png)

![Screenshot 2026-03-19 at 7.57.04 PM.png](Scalability/Screenshot_2026-03-19_at_7.57.04_PM.png)

![Screenshot 2026-03-19 at 7.57.43 PM.png](Scalability/Screenshot_2026-03-19_at_7.57.43_PM.png)

![Screenshot 2026-03-19 at 7.58.00 PM.png](Scalability/Screenshot_2026-03-19_at_7.58.00_PM.png)

- **The Stateful Way (Server-Side Sessions):** When a user logs in, the server creates a file or a record in its memory containing the user's details. It then sends a short, random string (a Session ID) back to the user's browser. When the user makes their next request, they send the Session ID. The server has to look up that ID in its memory to figure out who the user is. If the load balancer sends the user to a *different* server, that new server won't have the Session ID in its memory, and the user gets logged out.
- **The Stateless Way (JWT):** A JSON Web Token (JWT) solves this by packing the user's actual data (like their user ID, roles, and expiration time) directly into the token itself. When the user logs in, the server generates this JSON object, signs it cryptographically using a secret key, and sends it to the user.

**Why this makes the system stateless:** When the user makes their next request, they send the entire JWT. Because the token contains all the necessary information and is cryptographically signed, *any* server can verify the signature and read the user's identity instantly. The server doesn't need to remember anything from the previous request or look up a Session ID.

- **The Stateful Way (Local Disk):** Imagine a user uploads a new profile picture. If the application code saves that image directly to the server's local hard drive (e.g., `/var/www/uploads/profile.jpg`), that specific server is now unique. It holds state. If the load balancer routes the user's next request to Server B to view their profile, Server B will look at its own local disk, fail to find the image, and return a broken link. Furthermore, if Server A crashes and is replaced, that profile picture is gone forever.
- **The Stateless Way (Object Storage):** Services like Amazon S3, Google Cloud Storage, or Azure Blob Storage are massively scalable, standalone storage systems accessible via APIs. When a user uploads a file, your application server immediately forwards that file to S3. S3 returns a permanent URL for that file. The server then saves *only the URL* in your primary database, not the file itself.

![Screenshot 2026-03-19 at 7.58.17 PM.png](Scalability/Screenshot_2026-03-19_at_7.58.17_PM.png)

![Screenshot 2026-03-19 at 7.58.37 PM.png](Scalability/Screenshot_2026-03-19_at_7.58.37_PM.png)

![Screenshot 2026-03-19 at 7.59.35 PM.png](Scalability/Screenshot_2026-03-19_at_7.59.35_PM.png)

![Screenshot 2026-03-19 at 7.59.48 PM.png](Scalability/Screenshot_2026-03-19_at_7.59.48_PM.png)

**Common Sharding Strategies**

1. **Range-Based Sharding:** You divide data based on ranges of a value.

    ◦ *Example:* Users with IDs 1 to 1,000,000 go to Shard 1. IDs 1,000,001 to 2,000,000 go to Shard 2.

    ◦ *The Problem:* This often creates **hotspots**. If newer users are the most active on the platform, Shard 2 will be hammered with traffic while Shard 1 sits mostly idle.

2. **Hash-Based Sharding:** You run the shard key through a mathematical hash function (e.g., `hash(user_id) % 4_shards`).
    ◦ *Example:* The math guarantees that `user_id: 8943` always resolves to Shard 3.
    ◦ *The Benefit:* This distributes data and traffic very evenly across all servers, preventing hotspots.

3. **Directory/Geo-Based Sharding:** You maintain a dedicated lookup table that acts as a map, or you shard based on logical boundaries.

    ◦ *Example:* All users in Asia are routed to a database cluster in Mumbai; all users in Europe are routed to Frankfurt. This is excellent for reducing latency and complying with regional data laws.

**The Dark Side of Sharding**
Sharding is heavily considered a "last resort" in system design because it introduces immense complexity:

• **No More Simple Joins:** If you need to write a SQL `JOIN` query that combines data from User A (on Shard 1) and User B (on Shard 2), the database can no longer do it locally. The application has to pull data from both servers over the network and stitch it together in memory.

• **Resharding:** If your 4 shards fill up and you need to add a 5th, your hashing math changes (`hash(user_id) % 5`). Suddenly, millions of users are pointing to the wrong servers, and you have to migrate massive amounts of data without taking the system offline.

![Screenshot 2026-03-19 at 8.00.34 PM.png](Scalability/Screenshot_2026-03-19_at_8.00.34_PM.png)

![Screenshot 2026-03-19 at 8.00.57 PM.png](Scalability/Screenshot_2026-03-19_at_8.00.57_PM.png)

**1. The Cache-Aside Pattern (The Green Arrows)**

1. Check Cache: The application needs a user's profile. It asks Redis Node 1 for the data.
2. Cache Hit / Miss: If Redis has it (a "Hit"), it returns it immediately. If not, you get the "Miss" shown in the diagram.
3. Fallback to DB: The application queries the primary Database to get the profile.
4. Populate Cache: The application takes that database response and writes it into Redis. The next time anyone asks for that profile, it will be a Cache Hit.

**2. The Scaling Problem (Standard Hashing)**
When you have 3 Redis Nodes , the load balancer needs to know which node holds which cached item. The naive approach is a standard modulo hash: `hash(key) % 3`.
If `user_123` hashes to the number 10, then $10 mod 3 = 1$. The data goes to Node 1.
**The Catastrophe:** What happens when Black Friday hits, traffic spikes, and you need to add a 4th Redis Node?
Your math changes to `hash(key) % 4`. Now, $10 mod 4 = 2$. The load balancer looks for `user_123` on Node 2, gets a Cache Miss, and hammers the database. In fact, adding a single node changes the math for nearly *every* key, causing a massive wave of cache misses that can instantly crash your database.

**3. The Solution: Consistent Hashing**

Instead of a modulo array, imagine a logical ring of numbers from $0$ to $359$ (like degrees on a compass).

## 1. Place the Servers

You have three physical computers (Redis Node 1, Node 2, and Node 3) sitting in a data center. We need to assign them fixed spots on our painted circle.

- We feed the name **"Node 1"** into our magic machine. The machine spits out **90**. We walk over to the 90-degree mark on the circle and place a bucket there.
- We feed **"Node 2"** into the machine. It spits out **210**. We put a bucket at 210.
- We feed **"Node 3"** into the machine. It spits out **300**. We put a bucket at 300.

## 2. Place the Keys (The Data)

Now, hundreds of users are asking for different discount coupons. We need to figure out which bucket (server) should hold each coupon.

- A user asks for **"Coupon_A"**. We feed "Coupon_A" into our magic machine. The machine spits out **45**. We place "Coupon_A" at the 45-degree mark on the circle.
- A user asks for **"Coupon_B"**. We feed it into the machine. It spits out **250**. We place it at the 250-degree mark.

## The Clockwise Rule (How they connect)

**The rule is simple: Start at the coupon's location and walk clockwise around the circle until you bump into a bucket.**

- **"Coupon_A"** is at 45. If you walk clockwise, the very first bucket you hit is **Node 1** at 90. Therefore, Node 1 is responsible for storing Coupon A.
- **"Coupon_B"** is at 250. If you walk clockwise, the first bucket you hit is **Node 3** at 300. Node 3 stores Coupon B.

Here is the difference in the "blast radius" between the two methods:

## The Old Way (Standard Modulo Hashing)

If you have 300,000 cached coupons and 3 servers, you use `hash(key) % 3` to place them.
If you add a 4th server, the formula changes to `hash(key) % 4` for **every single key in the entire system**.
Because the denominator changed, the math output changes for almost everything. Suddenly, roughly 75% of your keys (225,000 coupons) are pointing to the wrong servers. **Result: 225,000 simultaneous cache misses.** Your database crashes instantly.

## The New Way (Consistent Hashing)

Imagine those same 300,000 coupons scattered around the ring, with 100,000 coupons going to each of the 3 servers.

You add Store 4 to the ring.
Store 4 lands in *one specific spot* (let's say, right between Store 1 and Store 3).

- Store 4 intercepts keys `a, b, c` (and let's say about 25,000 other keys that were previously walking toward Store 1).
- **Those 25,000 keys get a cache miss.** They hit the database.

To prevent even that 25% spike from hitting the database, engineers use a technique called **Cache Warming**. Before they allow the load balancer to send live user traffic to the newly added Store 4, they have a background script secretly pull `a, b, c` from the database and pre-load them into Store 4.

![Screenshot 2026-03-19 at 8.01.08 PM.png](Scalability/Screenshot_2026-03-19_at_8.01.08_PM.png)

![Screenshot 2026-03-19 at 8.01.20 PM.png](Scalability/Screenshot_2026-03-19_at_8.01.20_PM.png)

![Screenshot 2026-03-19 at 8.01.52 PM.png](Scalability/Screenshot_2026-03-19_at_8.01.52_PM.png)

![Screenshot 2026-03-19 at 8.02.24 PM.png](Scalability/Screenshot_2026-03-19_at_8.02.24_PM.png)

![Screenshot 2026-03-19 at 8.02.40 PM.png](Scalability/Screenshot_2026-03-19_at_8.02.40_PM.png)

![Screenshot 2026-03-19 at 8.02.54 PM.png](Scalability/Screenshot_2026-03-19_at_8.02.54_PM.png)

![Screenshot 2026-03-19 at 8.03.06 PM.png](Scalability/Screenshot_2026-03-19_at_8.03.06_PM.png)