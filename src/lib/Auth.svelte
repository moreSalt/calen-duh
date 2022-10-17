<script>
import { supabase } from '$lib/supabaseClient'
import { onMount } from 'svelte';

let loading = false
let email = "";
let sentOtp = false;
let token = "";
$: errorMessage = ""

const showError = async (message) => {
	console.log(message)
	errorMessage = message;
	new Promise(resolve => setTimeout(resolve, 5000));
	errorMessage = ""
}
// let signup = false;


const sendOtp = async () => {
	try {
		loading = false;
      const { data, error } = await supabase.auth.signInWithOtp({ email })
      if (error) throw error

	//   console.log(data)
	sentOtp = true;

    //   alert('Check your email for the login link!')
    } catch (error) {
    //   alert(error.error_description || error.message)
    //   console.log(error.error_description || error.message)
		await showError(error.error_description || error.message)
    } finally {
      loading = false
    }
}

</script>

<form
	class="flex flex-col gap-4 justify-center items-center h-screen"
	on:submit|preventDefault={sendOtp}
>
	<div class="flex flex-col gap-4 bg-base-300 p-6 rounded shadow-xl w-full max-w-sm">
		<h1 class="text-3xl font-bold">Calenduh</h1>
			{#if sentOtp}
				<p>Please check your email</p>
			
			{:else}
				<div class="form-control w-full">
						
					<label class="label" for="nunya">
						<span class="label-text">Email</span>
						<!-- {#if errorMessage} -->
							<span class="label-text-alt text-error">{errorMessage}</span>
						<!-- {/if} -->
					</label>

					<input
						type="email"
						placeholder="example@calenduh.xyz"
						class="input input-bordered input-sm w-full"
						bind:value={email}
						required
					/>
				</div>
				<input
					type="submit"
					class="btn w-full btn-sm btn-success btn-outline"
					value={loading ? 'Loading' : "Submit"}
					disabled={loading}
				/>
			{/if}
		<div>
	</div>
</form>
