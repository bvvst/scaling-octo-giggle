<script lang="ts">
	//@ts-nocheck
	import '../app.css';
	import { ethers } from 'ethers';
	import { onMount } from 'svelte';
	import { userAddr, authCookie } from '../store.ts';

	let addr;

	userAddr.subscribe((v) => (addr = v));
	let provider: string;
	let signer: string;

	let cookie;
	authCookie.subscribe((v) => (cookie = v));

	function getCookie(cookieName: string) {
		let cookie = {};
		document.cookie.split(';').forEach(function (el) {
			let [key, value] = el.split('=');
			cookie[key.trim()] = value;
		});
		return cookie[cookieName];
	}

	onMount(async () => {
		if (window.ethereum.isConnected()) {
			provider = new ethers.BrowserProvider(window.ethereum);

			const accounts = await window.ethereum.request({ method: 'eth_accounts' });
			if (accounts.length > 0) {
				signer = await provider.getSigner();
				console.log(signer);
				userAddr.set(accounts[0]);
			}
		}

		window.ethereum.on('disconnect', () => {
			userAddr.set(null);
		});

		authCookie.set(getCookie('auth'));
	});

	async function connectWallet() {
		if (typeof window.ethereum !== 'undefined') {
			try {
				const accounts = await window.ethereum.request({ method: 'eth_requestAccounts' });
				if (accounts.length > 0) {
					signer = await provider.getSigner();
					console.log(signer);
					userAddr.set(accounts[0]);
				}
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
				addr: addr
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
			authCookie.set(data.token);
		}
	}

	function logout() {
		document.cookie = 'auth' + '=; Max-Age=-99999999;';
		authCookie.set(null);
	}
</script>

<div class="p-4">
	<div class="flex gap-2 items-center">
		{#if addr}
			{#if cookie}
				<button on:click={logout}>
					<div class="retro flex">log out</div>
				</button>
				<p>Connected account: {addr}</p>
			{:else}
				<button on:click={login}>
					<div class="retro">login</div>
				</button>
			{/if}
		{:else}
			<button on:click={connectWallet}>
				<div class="retro">connect wallet</div>
			</button>
		{/if}
	</div>
	<slot />
</div>
