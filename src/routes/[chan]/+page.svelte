<script>
	//@ts-nocheck
	export let data;
	import { onMount, onDestroy } from 'svelte';
	import { createClient } from '@supabase/supabase-js';
	import { userAddr, authCookie } from '../../store';

	let addr;
	let cookie;

	userAddr.subscribe((v) => (addr = v));
	authCookie.subscribe((v) => (cookie = v));
	const sb = createClient(
		'https://lignorhoomxeyhyetgdf.supabase.co',
		import.meta.env.VITE_ANON_KEY
	);

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
		messages = await fetchMessages();
		const subscription = sb
			.channel('table-db-changes')
			.on(
				'postgres_changes',
				{
					event: 'INSERT',
					schema: 'public',
					table: 'messages',
					filter: 'channel=eq.' + data.channel
				},
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
			.eq('channel', data.channel)
			.order('created_at', { ascending: false })
			.limit(30);
		if (error) console.error(error);
		console.log(messages[0]);
		return messages;
	};

	const handleNewMessage = (payload) => {
		const newMessage = payload.new;
		console.log(newMessage.temp_id);

		const isDuplicate = messages.some((msg) => msg.temp_id == newMessage.temp_id);
		if (!isDuplicate) {
			messages = [newMessage, ...messages];
		}
	};

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
		let id = Math.floor(Math.random() * 9999999).toString();
		let newMessage = {
			channel: 1,
			content: message,
			created_at: 1,
			id: 1,
			temp_id: id,
			user_addr: addr
		};
		messages = [newMessage, ...messages];
		let content = message;
		message = '';

		if (content.trim().length > 0) {
			const res = await fetch('https://friendly-disco-production.up.railway.app/message', {
				method: 'POST',
				headers: {
					'Content-Type': 'application/json',
					Authorization: `Bearer ${cookie}`
				},
				body: JSON.stringify({
					content: content,
					channel: data.channel,
					temp_id: id
				})
			});

			if (res.status != 201) {
				console.log(res);
				messages = messages.filter((m) => m.temp_id != newMessage.temp_id);
			} else {
				console.log('message successful');
			}
		}
	}
</script>

<div class="flex flex-col mt-4">
	<div class="flex flex-col-reverse h-96 border p-4 border-black border-b-0 overflow-scroll">
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
		{#if cookie}
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
