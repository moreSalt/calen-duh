<script>
  import { supabase } from '$lib/supabaseClient'
  import { user } from '$lib/sessionStore'
	import { onMount } from 'svelte';
  let loading = false
  let username = ""


  async function signOut() {
    try {
      loading = true
      let { error } = await supabase.auth.signOut()
      if (error) throw error
    } catch (error) {
      // alert(error.message)
      console.error("sign out error", error)

    } finally {
      loading = false
    }
  }

  onMount(async () => {
    // await console.log(user)
    user.subscribe(value => {
      // console.log(value)
      username = value.email
    })
  })


</script>

<div class="navbar bg-base-300">
    <div class="navbar-start">

      <button class="btn btn-ghost normal-case text-xl" on:click={() => {
        console.log("Hey you found me!")
      }}>calenduh</button>
    </div>
    <div class="navbar-end gap-4">
      <label class="btn btn-primary btn-sm" for="my-modal-4">New event</label>
      <div class="dropdown dropdown-end">
        <!-- svelte-ignore a11y-no-noninteractive-tabindex -->
        <!-- svelte-ignore a11y-label-has-associated-control -->
        <label tabindex="0" class="btn btn-ghost btn-circle avatar">
          <div class="w-10 rounded-full">
            <!-- svelte-ignore a11y-missing-attribute -->
            <img src="https://source.boringavatars.com/beam/80" />
          </div>
        </label>
        <!-- svelte-ignore a11y-no-noninteractive-tabindex -->
        <ul tabindex="0" class="mt-3 p-2 shadow menu menu-compact dropdown-content rounded-box w-52 bg-base-300">
          <li>
            <!-- svelte-ignore a11y-missing-attribute -->
            <a class="truncate text-ellipsis">
              {username}
            </a>
          </li>
          <li><a href="https://github.com/moreSalt/calenduh" target="_blank">
            Github
            <svg xmlns="http://www.w3.org/2000/svg" class="h-4 w-4" fill="none" viewBox="0 0 24 24" stroke="currentColor"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M10 6H6a2 2 0 00-2 2v10a2 2 0 002 2h10a2 2 0 002-2v-4M14 4h6m0 0v6m0-6L10 14"></path></svg>
          </a></li>
          <!-- svelte-ignore a11y-click-events-have-key-events -->
          <!-- svelte-ignore a11y-missing-attribute -->
          <li on:click={signOut}><a>Logout</a></li>
        </ul>
      </div>
    </div>
  </div>