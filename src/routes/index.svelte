<script lang="ts">
	import { onMount } from 'svelte';
	import {TextInput, Button, DataTable, Grid, Row, Column, Tile} from "carbon-components-svelte";

	let ipfsNode1,ipfsNode2;
	let nodeId;
	let orbitdb1;
	let putted, added;
	let putWithLinks, addedWithLinks;

	let myAddressBook = "/orbitdb/zdpuAqSriPZbh2PWy1m5d7n1ZfzQPPmnGqMM8KGYxBHWgqXEt" //"myAddressBook";
	let myAddressBookDB;

	let allMyAddressFields = []
	let allMyContacts = []
	let currentAddress = {}
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
			// no ipfs saved to globalThis, so load it up
			const IPFSmodule = await import('../modules/ipfs-core/ipfs-core.js');
			const IPFS = IPFSmodule.default;
			ipfsNode1 = await IPFS.create({ config: config1, repo: './ipfs/1' });
			globalThis.ipfsNode1 = ipfsNode1;
		} else {
			ipfsNode1 = globalThis.ipfsNode1;
		}

		const identity = await ipfsNode1.id();
		nodeId = identity.id;

		const OrbitDBModul = await import('../modules/orbitdb-core/index.js');
		const createOrbitDB = OrbitDBModul.createOrbitDB;
		// const ipfsAccessController = OrbitDBModul.IPFSAccessController;
		orbitdb1 = await createOrbitDB({ ipfs: ipfsNode1 })

		// //2. Bobs DAL
		// const bobDAL = "bobDAL"
		// const bobDALDB = await orbitdb2.open(bobDAL, { type: 'keyvalue' })
		// await bobDALDB.put('NAME', "Bob Smith")
		// await bobDALDB.put('STREET', "The Address Downtown")
		// await bobDALDB.put('CITY', "Dubai")
		// const allBob = await bobDALDB.all()
		// console.log("allBob",allBob)

		//3. My address book


		// const bobsName = await bobDALDB.get("NAME")
		// let allAliceaddressbook = await myAddressBookDB.all()
		// console.log("allAliceaddressbook",allAliceaddressbook)

		//4. Bobs address book (has only his address
		// const myAddressBook = "myAddressBook"
		// const bobsAddressBookDB = await orbitdb2.open(myAddressBook, { type: 'keyvalue' })
		// await bobsAddressBookDB.put('MY-DAL', bobDALDB.address)
		// const allbobsAddressBookDB = await bobsAddressBookDB.all()
		// console.log("allbobsAddressBookDB",allbobsAddressBookDB)

		//Now Alice is connecting (scanning) Bobs address book (we need his address not his contacts!)
		//TODO who gets read access for bobs address book? only bob or everybody with a password or privatekey?
		// const alice2BobsDAL = await orbitdb1.open(bobDALDB.address, { type: 'keyvalue' })
		// const allalice2BobsDAL = await alice2BobsDAL.all()
		// console.log("allalice2BobsDAL",allalice2BobsDAL) //alice orbitdb now contains bobs address

		//we now put bobs dal book into alice addressbook and iterator over all entries in alice address book and print out all DALs
		// await myAddressBookDB.put(bobsName, bobDALDB.address)
		// allAliceaddressbook = await myAddressBookDB.all()
		// console.log("allAliceaddressbook",allAliceaddressbook)

		//now change bobs address (Bob is moving to a new address)
		// const newAddress = "Jumeirah Beach"
		// console.log("changing bobs address",newAddress)
		// await bobDALDB.put('STREET',newAddress)

		// for (let key in allAliceaddressbook) {
		// 	console.log("obj",allAliceaddressbook[key])
		// 	const dbAddr = allAliceaddressbook[key].value
		// 	console.log(`connecting to ${allAliceaddressbook[key].key}'s ${dbAddr} `)
		// 	const syncingDAL = await orbitdb1.open(dbAddr, { type: 'keyvalue' })
		// 	const allSyncingDAL = await syncingDAL.all()
		// 	console.log("allSyncingDAL",allSyncingDAL)
		// }

		return () => {
			console.log('the ipfs node is being stopped');
			ipfsNode1.stop();
			ipfsNode2.stop();
			globalThis.ipfsNode1 = null;
			globalThis.ipfsNode2 = null;
		};
	});

	const connect = async () => {
		console.log("connecting to ",myAddressBook)
		myAddressBookDB = await orbitdb1.open(myAddressBook, { type: 'keyvalue' })
		//await myAddressBookDB.put('MY-DAL', aliceDALDB.address)
		console.log("myAddressBookDB",myAddressBookDB)
		myAddressBook = myAddressBookDB.address
		console.log("myAddressBook now (address)",myAddressBook)

		const allContactDBs = await myAddressBookDB.all()
		allMyContacts = allContactDBs.map((it, i)=>{
			const newContact = it
			newContact.id = i
			return newContact
		})
		let i = 0
		for (let key in allContactDBs) {
			if(allContactDBs[key].key==="MY-DAL"){
				await loadMyDAL("MY-DAL")
			}
		}

	}
	const loadMyDAL = async (dal) => {
		const aliceDALDB = await orbitdb1.open(dal, { type: 'keyvalue' })
		await aliceDALDB.put('NAME', "Alice Brown")
		await aliceDALDB.put('CITY', "Istanbul")
		allMyAddressFields = await aliceDALDB.all()
	}

	const loadMyContactFrom = async (e) => {
		console.log("row clicked",e.detail.key)
		const dbAddr  = e.detail.value
		console.log(`connecting to ${dbAddr} `)
		const syncingDAL = await orbitdb1.open(dbAddr, { type: 'keyvalue' })
		const allSyncingDAL = await syncingDAL.all()
		console.log("currentAddress of "+e.detail.key,toObject(allSyncingDAL))
		currentAddress = toObject(allSyncingDAL)
	}

	function toObject(arr) {
		var rv = {};
		for (var i = 0; i < arr.length; ++i)
			rv[arr[i].key] = arr[i].value;
		return rv;
	}

	const disconnect = async () => {
		console.log("disconnecting ",myAddressBook)
		const res = await myAddressBookDB.close()
		console.log("myAddressBookDB",myAddressBookDB)
	}

</script>

<section>
	{#if nodeId}
		<h3>
			Node ID: {nodeId}
		</h3>
		<TextInput labelText="myAddressBook" placeholder="enter addressbook name or address" bind:value={myAddressBook}/>
		{#if !myAddressBookDB}
			<Button on:click={connect}>Connect to addressbook</Button>
		{:else}
			<Button on:click={disconnect}>Disconnect {myAddressBookDB.name}</Button>
		{/if}

		<p></p>
		<p></p>
		<p></p>
		<Grid>
			<Row><Column>My Address:</Column><Column></Column></Row>
			{#each allMyAddressFields as field}
				<Row><Column><h3>{field.key} {field.value}</h3></Column></Row>
			{/each}
			<Row><Column></Column><Column></Column></Row>
			<Row><Column></Column><Column></Column></Row>
			<Row><Column></Column><Column></Column></Row>
			<Row><Column></Column><Column></Column></Row>
		</Grid>
		<p></p>
		<p></p>

		<DataTable
				on:click:row={(e) => loadMyContactFrom(e)}
				headers={[
                    { key: "key", value: "name" },
                    { key: "value", value: "address" },
                  ]}
				bind:rows={allMyContacts}
		/>
		<p>&nbsp;</p>
		<Tile>
			{currentAddress?.NAME?currentAddress?.NAME:''}
			{currentAddress?.STREET?currentAddress?.STREET:''}
			{currentAddress?.CITY?currentAddress?.CITY:''}
		</Tile>

	{:else}
		Loading IPFS...
	{/if}
</section>