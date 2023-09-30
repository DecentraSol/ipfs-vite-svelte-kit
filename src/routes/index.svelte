<script lang="ts">
	import { onMount, onDestroy } from 'svelte';
	import { fromString as uint8ArrayFromString } from 'uint8arrays/from-string';
	import { UnixFS } from 'ipfs-unixfs';

	let ipfsNode1,ipfsNode2;;
	let nodeId;
	let putted, added;
	let putWithLinks, addedWithLinks;

	let helloWorld = {
		hello: 'world'
	};

	onMount(async () => {
		const config1 = {
			// Addresses: {
			// 	API: '/ip4/127.0.0.1/tcp/0',
			// 	Swarm: ['/ip4/0.0.0.0/tcp/0'],
			// 	Gateway: '/ip4/0.0.0.0/tcp/0'
			// },
			Bootstrap: [],
			Addresses: {
				Swarm: [
					// '/dns4/ipfs.le-space.de/tcp/4003/wss/p2p/12D3KooWALjeG5hYT9qtBtqpv1X3Z4HVgjDrBamHfo37Jd61uW1t',
					// '/dns6/ipfs.le-space.de/tcp/4003/wss/p2p/12D3KooWALjeG5hYT9qtBtqpv1X3Z4HVgjDrBamHfo37Jd61uW1t'
					// '/dns4/ws-star.discovery.libp2p.io/tcp/443/wss/p2p-websocket-star',
					// '/dns4/ipfs.infura.io/tcp/443/wss/p2p-webrtc-star',
					// '/p2p-circuit/ipfs/12D3KooWALjeG5hYT9qtBtqpv1X3Z4HVgjDrBamHfo37Jd61uW1t',
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

		const config2 = {
			Addresses: {
				Swarm: [
					// '/dns4/ipfs.le-space.de/tcp/4003/wss/p2p/12D3KooWALjeG5hYT9qtBtqpv1X3Z4HVgjDrBamHfo37Jd61uW1t',
					// '/dns6/ipfs.le-space.de/tcp/4003/wss/p2p/12D3KooWALjeG5hYT9qtBtqpv1X3Z4HVgjDrBamHfo37Jd61uW1t'
					// '/dns4/ws-star.discovery.libp2p.io/tcp/443/wss/p2p-websocket-star',
					// '/dns4/ipfs.infura.io/tcp/443/wss/p2p-webrtc-star',
					// '/p2p-circuit/ipfs/12D3KooWALjeG5hYT9qtBtqpv1X3Z4HVgjDrBamHfo37Jd61uW1t',
					'/dns6/ipfs.le-space.de/tcp/9091/wss/p2p-webrtc-star',
					'/dns4/ipfs.le-space.de/tcp/9091/wss/p2p-webrtc-star'
				],
			},
			Bootstrap: [],
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
			// no ipfs saved to globalThis, so load it up
			const IPFSmodule = await import('../modules/ipfs-core/ipfs-core.js');
			const IPFS = IPFSmodule.default;
			ipfsNode1 = await IPFS.create({ config: config1, repo: './ipfs/1' });
			ipfsNode2 = await IPFS.create({ config: config2, repo: './ipfs/2' });
			globalThis.ipfsNode1 = ipfsNode1;
		} else {
			ipfsNode1 = globalThis.ipfsNode1;
			ipfsNode2 = globalThis.ipfsNode2;
		}

		const identity = await ipfsNode1.id();
		nodeId = identity.id;

		const OrbitDBModul = await import('../modules/orbitdb-core/index.js');
		const createOrbitDB = OrbitDBModul.createOrbitDB;
		// const ipfsAccessController = OrbitDBModul.IPFSAccessController;
		const orbitdb1 = await createOrbitDB({ ipfs: ipfsNode1 })
		const orbitdb2 = await createOrbitDB({ ipfs: ipfsNode2 })

		//1. Create my own addressDB and receive a DAL (decentralized address link)
		const aliceDAL = "aliceDAL"
		const aliceDALDB = await orbitdb1.open(aliceDAL, { type: 'keyvalue' })
		await aliceDALDB.put('NAME', "Nico Krause")
		await aliceDALDB.put('CITY', "Eggenfelden")
		const allAlice = await aliceDALDB.all()
		console.log("allAlice",allAlice)

		//2. Bobs DAL
		const bobDAL = "bobDAL"
		const bobDALDB = await orbitdb2.open(bobDAL, { type: 'keyvalue' })
		await bobDALDB.put('NAME', "Irina Krause")
		await bobDALDB.put('STREET', "Mashinaya ul. 1b/1")
		await bobDALDB.put('CITY', "Jekaterinburg")
		const allBob = await bobDALDB.all()
		console.log("allBob",allBob)

		//3. My address book
		const aliceAddressBook = "myAddressBook"
		const aliceAddressBookDB = await orbitdb1.open(aliceAddressBook, { type: 'keyvalue' })
		await aliceAddressBookDB.put('MY-DAL', aliceDALDB.address)
		const bobsName = await bobDALDB.get("NAME")
		let allAliceaddressbook = await aliceAddressBookDB.all()
		console.log("allAliceaddressbook",allAliceaddressbook)

		//4. Bobs address book (has only his address
		// const myAddressBook = "myAddressBook"
		const bobsAddressBookDB = await orbitdb2.open(aliceAddressBook, { type: 'keyvalue' })
		await bobsAddressBookDB.put('MY-DAL', bobDALDB.address)
		const allbobsAddressBookDB = await bobsAddressBookDB.all()
		console.log("allbobsAddressBookDB",allbobsAddressBookDB)

		//Now Alice is connecting (scanning) Bobs address book (we need his address not his contacts!)
		//TODO who gets read access for bobs address book? only bob or everybody with a password or privatekey?
		const alice2BobsDAL = await orbitdb1.open(bobDALDB.address, { type: 'keyvalue' })
		const allalice2BobsDAL = await alice2BobsDAL.all()
		console.log("allalice2BobsDAL",allalice2BobsDAL) //alice orbitdb now contains bobs address

		//we now put bobs dal book into alice addressbook and iterator over all entries in alice address book and print out all DALs
		await aliceAddressBookDB.put(bobsName, bobDALDB.address)
		allAliceaddressbook = await aliceAddressBookDB.all()
		console.log("allAliceaddressbook",allAliceaddressbook)

		//now change bobs address (Bob is moving to a new address)
		const newAddress = "Chevchenko ul. 20"
		console.log("changing bobs address",newAddress)
		await bobDALDB.put('STREET',newAddress)

		for (let key in allAliceaddressbook) {
			console.log("obj",allAliceaddressbook[key])
			const dbAddr = allAliceaddressbook[key].value
			console.log(`connecting to ${allAliceaddressbook[key].key}'s ${dbAddr} `)
			const syncingDAL = await orbitdb1.open(dbAddr, { type: 'keyvalue' })
			const allSyncingDAL = await syncingDAL.all()
			console.log("allSyncingDAL",allSyncingDAL)
		}

		return () => {
			console.log('the ipfs node is being stopped');
			ipfsNode1.stop();
			ipfsNode2.stop();
			globalThis.ipfsNode1 = null;
			globalThis.ipfsNode2 = null;
		};
	});
</script>

<svelte:head>
	<title>Home</title>
</svelte:head>

<section>
	{#if nodeId}
		<h3>
			Node ID: {nodeId}
		</h3>
		<h2>
			IPFS loaded in a Vite app, <strong>right?!</strong>
		</h2>
		{#if putted}ipfs.dag.put({JSON.stringify(helloWorld)}, {"{ storeCodec: 'dag-pb' }"})
			<br />
			CID verion0: {putted.toV0().toString()}
			<br />
			CID verion1: {putted.toString()}
		{/if}<br /><br />
		{#if added}ipfs.add({JSON.stringify(helloWorld)})<br />CID v0: {added.cid
				.toV0()
				.toString()}{/if}
		<br /><br />
		{#if added}ipfs.add(data) >> ipfs.cat(data):<br />

			<a target="_blank" href="https://dweb.link/api/v0/cat?arg={added.cid.toV0().toString()}"
				>{added.cid.toV0().toString()}</a
			>
			<br /> Check that they are the same:
			<b> {putted.toV0().toString() === added.cid.toV0().toString()}</b>
		{/if}
		<br /><br />
		{#if putWithLinks}Explore Linked Data:<br /><a
				target="_blank"
				href="https://explore.ipld.io/#/explore/{putWithLinks.toV0().toString()}"
				>{putWithLinks.toV0().toString()}</a
			>

			<br />DWeb Site:<br /><a
				target="_blank"
				href="https://{putWithLinks.toV1().toString()}.ipfs.dweb.link/"
				>https://[cid].ipfs.dweb.link</a
			><br /><br />

			CloudFlare:<br />
			<a target="_blank" href="https://{putWithLinks.toV1().toString()}.ipfs.cf-ipfs.com/"
				>https://[cid].ipfs.cf-ipfs.com</a
			><br /><br />
		{/if}

		{#if addedWithLinks}Explore Add with Links:<br /><a
				target="_blank"
				href="ipfs://bafybeiftcyj7gao3kykwic743wuulwpzkeat4nfmzapbgz5mxsfxsnpbtu/#/explore/{addedWithLinks.cid
					.toV0()
					.toString()}">{addedWithLinks.cid.toV0().toString()}</a
			>

			<br />DWeb Site:<br /><a
				target="_blank"
				href="https://{addedWithLinks.cid.toV1().toString()}.ipfs.dweb.link/">View Data</a
			><br />Mirror:<br />

			<a target="_blank" href="https://{addedWithLinks.cid.toV1().toString()}.ipfs.cf-ipfs.com/"
				>View Mirror Data</a
			><br /><br />
		{/if}
	{:else}
		Loading IPFS...
	{/if}
</section>

<style>
	section {
		display: flex;
		flex-direction: column;
		justify-content: center;
		align-items: center;
		flex: 1;
	}

	h1 {
		width: 100%;
	}
</style>
