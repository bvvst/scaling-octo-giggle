<script>
	//@ts-nocheck

	import { ethers } from 'ethers';
	import { onMount, onDestroy } from 'svelte';
	import { createClient } from '@supabase/supabase-js';

	const sb = createClient(
		'https://lignorhoomxeyhyetgdf.supabase.co',
		import.meta.env.VITE_ANON_KEY
	);

	let provider;
	let signer;
	let account;
	let authCookie;

	let messages = [];

	function getCookie(cookieName) {
		let cookie = {};
		document.cookie.split(';').forEach(function (el) {
			let [key, value] = el.split('=');
			cookie[key.trim()] = value;
		});
		return cookie[cookieName];
	}

	onMount(async () => {
		console.log(import.meta.env.VITE_ANON_KEY);
		if (window.ethereum.isConnected()) {
			provider = new ethers.BrowserProvider(window.ethereum);

			const accounts = await window.ethereum.request({ method: 'eth_accounts' });
			if (accounts.length > 0) {
				signer = await provider.getSigner();
				console.log(signer);
				account = accounts[0];
			}
		}

		authCookie = getCookie('auth');

		messages = await fetchMessages();
		const subscription = sb
			.channel('table-db-changes')
			.on(
				'postgres_changes',
				{ event: 'INSERT', schema: 'public', table: 'messages' },
				handleNewMessage
			)
			.subscribe();
		return () => {
			subscription.unsubscribe();
		};
	});

	onDestroy(() => {
		sb.removeAllChannels();
	});

	const fetchMessages = async () => {
		let { data: messages, error } = await sb
			.from('messages')
			.select('*')
			.order('created_at', { ascending: false })
			.limit(10);
		if (error) console.error(error);
		console.log(messages);
		return messages;
	};

	const handleNewMessage = (payload) => {
		const newMessage = payload.new;
		messages = [newMessage, ...messages];
	};

	async function connectWallet() {
		if (typeof window.ethereum !== 'undefined') {
			try {
				await window.ethereum.request({ method: 'eth_requestAccounts' });
				signer = await provider.getSigner();
				account = await signer.getAddress();
			} catch (error) {
				console.error('Error connecting to MetaMask:', error);
			}
		} else {
			alert('MetaMask is not installed. Please install it and try again.');
		}
	}

	async function fetchNonce() {
		const response = await fetch('https://lignorhoomxeyhyetgdf.functions.supabase.co/nonce', {
			method: 'POST',
			headers: {
				'Content-Type': 'application/json',
				Authorization: `Bearer ${import.meta.env.VITE_ANON_KEY}`
			},
			body: JSON.stringify({
				addr: account
			})
		});
		const data = await response.json();
		return data.nonce;
	}

	async function login() {
		//get nonce and sign message
		const nonce = await fetchNonce();
		const signature = await signer.signMessage(
			`Sign in to Terminally Online by signing this message. Nonce: ${nonce}`
		);

		console.log(signature);

		// post signature
		const response = await fetch('https://lignorhoomxeyhyetgdf.functions.supabase.co/login', {
			method: 'POST',
			headers: {
				'Content-Type': 'application/json',
				Authorization: `Bearer ${import.meta.env.VITE_ANON_KEY}`
			},
			body: JSON.stringify({
				signature: signature,
				nonce: nonce
			})
		});

		if (response.status == 200) {
			const data = await response.json();
			console.log('successfully logged in: ', data.token);
			document.cookie = `auth=${data.token}; SameSite=Lax; Path=/; Secure; Max-Age=604800`;
			authCookie = data.token;
		}
	}

	function shortenAddr(str) {
		if (str.length < 6) {
			throw new Error('Input string must be at least 6 characters long.');
		}

		const firstTwoChars = str.slice(0, 2);
		const lastFourChars = str.slice(-4);
		const ellipse = '...';

		return firstTwoChars + ellipse + lastFourChars;
	}

	let message = '';

	async function handleSubmit(event) {
		event.preventDefault(); // Prevents default form submission behavior
		if (message.trim().length > 0) {
			const response = await fetch('https://friendly-disco-production.up.railway.app/message', {
				method: 'POST',
				headers: {
					'Content-Type': 'application/json',
					Authorization: `Bearer ${authCookie}`
				},
				body: JSON.stringify({
					content: message
				})
			});
			console.log(response);

			message = '';
		}
	}

	function logout() {
		document.cookie = 'auth' + '=; Max-Age=-99999999;';
		authCookie = null;
	}
</script>

<div class="p-4">
	<div class="flex gap-2 items-center">
		{#if account}
			{#if authCookie}
				<button on:click={logout}>
					<div class="retro flex">log out</div>
				</button>
				<p>Connected account: {account}</p>
			{:else}
				<button on:click={login}>
					<div class="retro">login</div>
				</button>
			{/if}
		{:else}
			<button on:click={connectWallet}>
				<div class="retro">connect</div>
			</button>
		{/if}
	</div>

	<div class="flex flex-col mt-4">
		<div class="flex flex-col-reverse h-60 border p-4 border-black border-b-0 overflow-scroll">
			{#each messages as m}
				<div>
					<span class="font-medium text-gray-500 mr-3">
						{shortenAddr(m.user_addr)}
					</span>

					{m.content}
				</div>
			{/each}
		</div>
		<div class="h-20 border border-black p-4">
			{#if authCookie}
				<form class="flex gap-2" on:submit={handleSubmit}>
					<input
						class="border border-black flex w-full focus:outline-none"
						type="text"
						bind:value={message}
						placeholder="Type your message here"
					/>
					<button type="submit">
						<div class="retro">send</div>
					</button>
				</form>
			{/if}
		</div>
	</div>
</div>
