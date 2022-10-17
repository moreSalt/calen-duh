<script>
	import { user } from '$lib/sessionStore';
	import { supabase } from '$lib/supabaseClient';
	import Auth from '$lib/Auth.svelte';
	import Calendar from '$lib/Calendar.svelte';
	import { onMount } from 'svelte';

	// onMount(async () => {
	// 	const {data , error } = await supabase.auth.getUser()
	// 	// await user.set(data.user);
	// 	await console.log(data)
	// 	user.subscribe(value => {
	// 		console.log(value)
    // 	})
	// });
	// user.set(supabase.auth.user())

	onMount(async () => {
		const { data, error} = await supabase.auth.getUser()
		if (error) {
			console.log ("error", error)
		}
		await user.set(data.user)
		
	})

    supabase.auth.onAuthStateChange((event, session) => {
		if (event === "SIGNED_IN") {
			user.set(session.user);
		} else {
			// https://github.com/supabase/supabase/discussions/3468#discussioncomment-2742352
			user.set(false)
		}
    });
</script>

{#if $user}
    <!-- <Profile /> -->
	<Calendar />
{:else}
    <Auth />
{/if}
