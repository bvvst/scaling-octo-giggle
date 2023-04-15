<script lang="ts">
	import { goto } from '$app/navigation';

	let room: string;

	function goToRoom() {
		setTimeout(() => goto('/' + room), 0);
	}
</script>

<div class="flex flex-col mt-4">
	<div class="flex gap-2">
		<input
			class="border border-black flex focus:outline-none"
			type="text"
			bind:value={room}
			placeholder="room"
		/>
		<button on:click={goToRoom} type="submit">
			<div class="retro">go</div>
		</button>
	</div>
</div>
