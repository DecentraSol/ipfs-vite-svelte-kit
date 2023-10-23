<script lang="ts">
	import { onMount } from 'svelte';


	let ipfsNode1;
	let orbitdb1;
	let db;
	let nodeId;
	let count;
	let IPFSAccessController

	onMount(async () => {
		const config1 = {
			Bootstrap: [],
			Addresses: {
				Swarm: [
					'/dns6/ipfs.le-space.de/tcp/9091/wss/p2p-webrtc-star',
					'/dns4/ipfs.le-space.de/tcp/9091/wss/p2p-webrtc-star'
				],
			},
			Discovery: {
				MDNS: {
					Enabled: true,
					Interval: 0
				}
			}
		}

		// In Svelte, a hot module refresh can cause lockfile problems
		// so we assign the ipfs node to globalThis to avoid lock file issues
		if (!globalThis.ipfsNode) {
			const IPFSmodule = await import('../modules/ipfs-core/ipfs-core.js');
			const IPFS = IPFSmodule.default;
			ipfsNode1 = await IPFS.create({ config: config1, repo: './ipfs/1' });

			console.log("ipfsNode1",ipfsNode1)
			//
			// ipfsNode1.on('peer:discovery', (peer) => {
			//   console.log('discovered', peer)
			// })
			//
			// ipfsNode1.on('peer:connect', async (peer) => {
			//   console.log('connected', peer)
			//
			// 	ipfsNode1.swarm.peers().then(peers => console.log('current peers connected: ', peers))
			// })
			globalThis.ipfsNode1 = ipfsNode1;
		} else {
			ipfsNode1 = globalThis.ipfsNode1;
		}

		const identity = await ipfsNode1.id();
		nodeId = identity.id;

		const OrbitDBModul = await import('../modules/orbitdb-core/index.js');
		const createOrbitDB = OrbitDBModul.createOrbitDB;
		IPFSAccessController = OrbitDBModul.IPFSAccessController;
		orbitdb1 = await createOrbitDB({ ipfs: ipfsNode1 })
		const dbAddress = "/orbitdb/zdpuAynrr9SX7Q8xL7ZmJw2eKZePdoNE3AXZqLjz7gJEAJin4" //"counter" //"counter"
		db = await orbitdb1.open(dbAddress, {
			type: 'keyvalue',
			sync: true,
			indexBy: 'id',
			create: true,
			//overwrite: false,
			public: true,
			AccessController: IPFSAccessController({ write: ['*'] })})
		console.log("db",db.events)
		db.events.on("replicated", async (dbAddress, count, newFeed, d) => {
			console.log("replicated - loading posts from db");
			// log.msg("dbAddress", dbAddress);
			// log.msg("count", count);
			// log.msg("feed", newFeed);

		});

		db.events.on("write", async (hash, entry, heads) => {
			console.log("orbit db wrote to ipfs" + hash, entry);
			// this.addPostToStore(entry);
		});

		// When the database is ready (ie. loaded), display results
		db.events.on("ready", (dbAddress, feedReady) => {
			console.log("database ready " + dbAddress, feedReady);
			// if(this.feed !== undefined) this.feed.all.map(this.addPostToStore);
			// else log.danger('feed is still undefined although ready')
		});

		db.events.on("replicate.progress", async (dbAddress, hash, obj) => {
			console.log("replicate.progress", dbAddress, hash);
			// log.msg('post subject',obj.payload.value.subject)
			// log.msg('post body',obj.payload.value.body)
			// if(obj.payload.op==="DEL"){
			// 	const entryHash = obj.payload.value
			// 	for (let i = 0; i < this.posts.length; i++) {
			// 		log.msg("iterating through posts>", this.posts[i].hash, this.posts[i].address);
			// 		if (this.posts[i].hash === entryHash) {
			// 			log.success("removed post from store because it was deleted on another node",entryHash);
			// 			this.posts = [] //empty store because it gets reloaded anyways
			// 		}
			// 	}
			// }
		});

		console.log("db.address",db.address)
		// await db.close()
		const dbReopened = await orbitdb1.open(db.address)

		console.log("dbReopened",dbReopened)
		count = await db.get('count')
		if(!count){
			await db.put('count', 0)
			count = await db.get('count')
		}
		const all = await db.all()
		console.log("all",all)

		return () => {
			console.log('the ipfs node is being stopped');
			ipfsNode1.stop();
			globalThis.ipfsNode1 = null;
		};
	});
</script>

<section>
	{#if nodeId}
		<h3>
			Node ID: {nodeId}
		</h3>
		hello our count is <div on:click={
			async () => {
				console.log("count is",count)
				await db.put('count', ++count)
			}}>{count}</div>

	{:else}
		Loading IPFS...
	{/if}
</section>