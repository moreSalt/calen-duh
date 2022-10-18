<script>
    import Nav from '$lib/Nav.svelte';
    import { supabase } from '$lib/supabaseClient'
    import c from "calendar";
	import { onMount } from 'svelte';

    // DAYS NAME ARR
    const days = [
        "Sunday",
        "Monday",
        "Tuesday",
        "Wednesday",
        "Thursday",
        "Friday",
        "Saturday",
    ];

    // MONTHS NAME ARR
    const months = [
        "January",
        "February",
        "March",
        "April",
        "May",
        "June",
        "July",
        "August",
        "September",
        "October",
        "November",
        "December",
    ];

    // EVENTS ARR
    let events = [];

    let devDays = [];

    // Get current month and date
    let currentY = `${new Date().getUTCFullYear()}`;
    let currentM = `${new Date().getMonth()}`;
    // Current calendar page
    let currentPage = [];
    
    // Update the calendarPage
    function updateDate() {
        const cal = new c.Calendar(); // weeks start on Sunday by default
        const m = cal.monthDates(
            parseInt(currentY),
            parseInt(currentM),
            function (d) {
                return (" " + d.getDate()).slice(-2);
            }
        );

        const tempMonth = parseInt(currentM);
        const tempDates = [];
        for (let i = 0; i < m.length; i++) {
            for (let j = 0; j < 7; j++) {
                tempDates.push(m[i][j]);
            }
        }
        devDays = [...tempDates];
        let index = 0;


        currentPage = tempDates.map((i) => {
            const obj = {
                // WHY DAY NEEDS THAT TRIM IDK
                day: i < 10 ? `0` + `${i}`.trim() : `${i}`,
                year: parseInt(currentY),
            };
            
            // Month logic
            const month = i > 20 && index < 10
                ? tempMonth : i < 10 && index > 20
                ? tempMonth + 2
                : tempMonth + 1;
            obj.month = parseInt(month) < 10 ? `0${month}` : `${month}`;

            obj.events = events.filter((e) =>
                e.start.includes(`${obj.year}-${obj.month}-${obj.day}T`)
            );
            index++;
            return obj;
        });

    }



    // Variable that populate calendar

    // 3AM wtf
    for (let i = 0; i < events.length; i++) {
        const e = events[i];
    }

    // Event form class, why i didnt do a func idk
    class EventForm {
        constructor() {
            // this.id = crypto.randomUUID();
            this.id = -1;
            this.name = "";
            this.start = "";
            this.end = ""; //2022-10-11T19:00"
            this.color = "primary";
            this.location = "";
            this.details = "";
        }
    }

    let eventForm = new EventForm();

    // Validation before creatng/updating
    async function save() {

        if (eventForm.name && eventForm.start) {
            const ev = {...eventForm};
            ev.start+=":00.000"

            await console.log(ev)
            // end time before start time logic
            if (ev.end) {
                ev.end+=":00.000"

                const st = new Date(ev.start+"Z");
                const en = new Date(ev.end+"Z");
                await console.log(st.getTime(), en.getTime())
                if (st.getTime() >= en.getTime()) {
                    return;
                }
            } 
            // ev.end = ev.end ? ev.end+=":00.000" : ev.end;

            await add(ev);
            await fetchDB();
            await updateDate();
            document.getElementById('my-modal-4').checked = false;
            currentSidebar = {...eventForm}

        } else {
            console.log("missing name or start")
        }
    }

    // Duplicate an event
    async function dup() {
        try {
            const temp = { ...currentSidebar };
            temp.name += " (copy)";
            delete temp.id
            eventForm = temp;
            const { error } = await supabase
                .from('events')
                .insert(temp)
            
            if (error) {
                throw error
            }

            await fetchDB()
            await updateDate()
        } catch (error) {
            console.log(error)
        }
    }

    // Delete an event
    async function del() {
        try {
            const { error } = await supabase
                .from('events')
                .delete()
                .eq('id', currentSidebar.id)

            // Fetch db
            await fetchDB()
            // Update current month
            await updateDate()
            // Reset event form
            eventForm = new EventForm();
        } catch (error) {
            console.error(error)
        }
    }

    // SIDEBAR
    let currentSidebar = {
        // id: 0,
        name: "",
        start: "",
        end: "", //2022-10-11T19:00"
        color: "primary",
        location: "",
        details: "",
    };

    // Fetch DB
    async function fetchDB() {
        try {
            // const { data, error } = await supabase.from('events').select().gte('start', '2000-01-01T00:00:00').lte('start', '2023-01-01T00:00:00');
            const { data, error } = await supabase
                .from('events')
                .select()
                .gte('start', `${currentPage[0].year}-${currentPage[0].month}-${currentPage[0].day}T00:00:00`)
                .lte('start', `${currentPage[currentPage.length-1].year}-${currentPage[currentPage.length-1].month}-${currentPage[currentPage.length-1].day}T23:59:59`);
            // const { data, error } = await supabase.from('events').select().gte('start', `${currentY}-${currentM < 10 ? "0" + currentM : currentM}-`).lte('start', '2023-01-01T00:00:00');
            if (error) {
                throw error;
            }
            events = [...data].map(e => {
                e.start = e.start.split(':').length > 1 ? e.start.split(':').slice(0,2).join(':') : e.start
                e.end = e.end && e.end.split(':').length > 1 ? e.end.split(':').slice(0,2).join(':') : e.end
                return e;
            });
            events.sort((a, b) => {
                let fa = a.start,
                    fb = b.start;

                if (fa < fb) {
                    return -1;
                }
                if (fa > fb) {
                    return 1;
                }
                return 0;
            });
        } catch (error) {
            console.error("error fetch calendar db", error)
        }
    }

    // Add or edit db
    async function add (event) {
        try {
            // 3AM LOGIC BABY
            const id = event.id;
            if (id < 0) {
                delete event.id;
            }
            // event = Object.keys(event).filter(e => event[e])
            const keys = Object.keys(event)
            for (let i = 0; i < keys.length; i++) {
                if (!event[keys[i]]) {
                    delete event[keys[i]]
                }
            }
            const { error } = id >= 0 ? await supabase.from('events').update(event).eq('id', event.id) : await supabase.from('events').insert(event)
            if (error) throw error;
            return true;
        } catch (error) {
            console.error(error)
            return false
        }
    }

    onMount(async () => {
        await updateDate();

        await fetchDB()
        await updateDate();
    })
</script>

<div class="drawer drawer-end">
    <input id="my-drawer-3" type="checkbox" class="drawer-toggle" />
    <div class="drawer-content flex flex-col">
        <!-- Navbar -->

        <!-- Page content here -->
        <!-- NAV -->
        <Nav />
        <!-- CALENDAR -->

        <div class=" p-2 h-full pb-12 flex flex-col gap-4">
            <!-- MONTH - YEAR -->
            <div class="col-span-full flex justify-center gap-4">
                <select
                    class="select select-bordered select-sm w-xs"
                    bind:value={currentM}
                    on:change={async () => {
                        await updateDate()
                        await fetchDB()
                        await updateDate()
                    }}
                >
                    <option value="0">January</option><option value="1"
                        >February</option
                    ><option value="2">March</option><option value="3"
                        >April</option
                    ><option value="4">May</option><option value="5"
                        >June</option
                    ><option value="6">July</option><option value="7"
                        >August</option
                    ><option value="8">September</option><option value="9"
                        >October</option
                    ><option value="10">November</option><option value="11"
                        >December</option
                    >
                </select>

                <select
                    class="select select-bordered select-sm w-xs"
                    bind:value={currentY}
                    on:change={async () => {
                        await updateDate()

                        await fetchDB()
                        await updateDate()
                    }}
                >
                    <option value="2020">2020</option><option value="2021"
                        >2021</option
                    ><option value="2022">2022</option><option value="2023"
                        >2023</option
                    ><option value="2024">2024</option>
                    <!-- <option value="0">January</option><option value="1">February</option><option value="2">March</option><option value="3">April</option><option value="4">May</option><option value="5">June</option><option value="6">July</option><option value="7">August</option><option value="8">September</option><option value="9">October</option><option value="10">November</option><option value="11">December</option>    -->
                </select>
            </div>

            <!-- Days of the Week -->
            <div class="col-span-full grid grid-cols-7 max-h-fit">
                <p class="text-center hidden md:inline-block h-fit">Sunday</p>
                <p class="text-center hidden md:inline-block h-fit">Monday</p>
                <p class="text-center hidden md:inline-block h-fit">Tuesday</p>
                <p class="text-center hidden md:inline-block h-fit">
                    Wednesday
                </p>
                <p class="text-center hidden md:inline-block h-fit">Thursday</p>
                <p class="text-center hidden md:inline-block h-fit">Friday</p>
                <p class="text-center hidden md:inline-block h-fit">Saturday</p>
            </div>

            <!-- CALENDAR DAYS -->
            <div class="grid grid-cols-1 md:grid md:grid-cols-7 grow">
                {#each currentPage as date, i}
                    <div
                        class="rounded-sm hover:border-secondary group p-2"
                        class:text-slate-500={(i < 7 &&
                            parseInt(date.day) > 20) ||
                            (i > 25 && parseInt(date.day) < 7)}
                    >
                        <h3
                            class="font-bold text-2xl w-fit group-hover:text-secondary group-hover:underline text-center pb-2 text-center"
                        >
                            {date.day}
                        </h3>
                        
                        <!-- EVENTS -->
                        <div class="menu">
                            {#each date.events as event, j}
                            <!-- svelte-ignore a11y-click-events-have-key-events -->
                            <!-- LEAVING IN AS WILL BE CHANGING THIS -->
                            <!-- {#if j < 2} -->
                            {#if true}
                                <label
                                    class="
                                gap-2
                                text-md
                                truncate 
                                text-ellipsis
                                rounded
                                hover:bg-neutral
                                hover:bg-opacity-25
                                hover:text-white
                                hover:pointer
                                cursor-pointer
                                max-w-full
                                hover:pl-2
                            "
                                    for="my-drawer-3"
                                    on:click={() => {
                                        eventForm = event;
                                        currentSidebar = event;
                                    }}
                                >
                                    <div
                                        class="
                                        badge
                                        badge-xs 
                                        badge-secondary
                                    "
                                        class:badge-primary={event.color ===
                                            "primary"}
                                        class:badge-secondary={event.color ===
                                            "secondary"}
                                        class:badge-accent={event.color ===
                                            "accent"}
                                        class:badge-info={event.color ===
                                            "info"}
                                        class:badge-success={event.color ===
                                            "success"}
                                        class:badge-warning={event.color ===
                                            "warning"}
                                        class:badge-error={event.color ===
                                            "error"}
                                    />
                                    {event.name}
                                </label>
                            {:else if j == 2}
                                <div
                                    class="
                                    gap-2
                                    text-md
                                    truncate 
                                    text-ellipsis
                                    rounded
                                    hover:bg-neutral
                                    hover:bg-opacity-25
                                    hover:text-white
                                    hover:pointer
                                    cursor-pointer
                                    h-fit
                                    hover:pl-2
                                    "
                                >
                                    + {date.events.length - 2} more
                            </div>

                            {/if}
                        {/each}
                        </div>
                    </div>
                {/each}
            </div>
        </div>
    </div>
    <div class="drawer-side">
        <label for="my-drawer-3" class="drawer-overlay" />

        <!-- SIDEBAR CONTENT -->
        <div
            class="flex flex-col justify-between p-4 overflow-y-auto w-80 bg-base-100 gap-4"
        >
            <div class="gap-4 flex flex-col grow">
                <!-- TITLE -->
                <h3 class="card-title gap-2 flex">
                    <div
                        class="badge badge-lg"
                        class:badge-primary={currentSidebar.color === "primary"}
                        class:badge-secondary={currentSidebar.color ===
                            "secondary"}
                        class:badge-accent={currentSidebar.color === "accent"}
                        class:badge-info={currentSidebar.color === "info"}
                        class:badge-success={currentSidebar.color === "success"}
                        class:badge-warning={currentSidebar.color === "warning"}
                        class:badge-error={currentSidebar.color === "error"}
                    />
                    {currentSidebar.name}
                </h3>

                <!-- Start date -->
                <div class="gap-2 flex">
                    <svg
                        xmlns="http://www.w3.org/2000/svg"
                        class="h-6 w-6"
                        fill="none"
                        viewBox="0 0 24 24"
                        stroke="currentColor"
                        ><path
                            stroke-linecap="round"
                            stroke-linejoin="round"
                            stroke-width="2"
                            d="M8 7V3m8 4V3m-9 8h10M5 21h14a2 2 0 002-2V7a2 2 0 00-2-2H5a2 2 0 00-2 2v12a2 2 0 002 2z"
                        /></svg
                    >
                    {currentSidebar.start}
                </div>

                <!-- End date -->
                <div class="gap-2 flex">
                    <svg
                        xmlns="http://www.w3.org/2000/svg"
                        class="h-6 w-6"
                        fill="none"
                        viewBox="0 0 24 24"
                        stroke="currentColor"
                        ><path
                            stroke-linecap="round"
                            stroke-linejoin="round"
                            stroke-width="2"
                            d="M8 7V3m8 4V3m-9 8h10M5 21h14a2 2 0 002-2V7a2 2 0 00-2-2H5a2 2 0 00-2 2v12a2 2 0 002 2z"
                        /></svg
                    >
                    {currentSidebar.end || "none"}
                </div>

                <!-- location -->
                <div class="gap-2 flex">
                    <svg
                        xmlns="http://www.w3.org/2000/svg"
                        class="h-6 w-6"
                        fill="none"
                        viewBox="0 0 24 24"
                        stroke="currentColor"
                    >
                        <path
                            stroke-linecap="round"
                            stroke-linejoin="round"
                            stroke-width="2"
                            d="M17.657 16.657L13.414 20.9a1.998 1.998 0 01-2.827 0l-4.244-4.243a8 8 0 1111.314 0z"
                        />
                        <path
                            stroke-linecap="round"
                            stroke-linejoin="round"
                            stroke-width="2"
                            d="M15 11a3 3 0 11-6 0 3 3 0 016 0z"
                        />
                    </svg>
                    {currentSidebar.location || "none"}
                </div>

                <!-- Details -->
                <textarea
                    name=""
                    id=""
                    class="textarea textarea-bordered h-full"
                    disabled>{currentSidebar.details}</textarea
                >
            </div>

            <div class="flex gap-2 justify-end">
                <!-- svelte-ignore a11y-click-events-have-key-events -->
                <label
                    class="btn btn-ghost btn-circle text-warning"
                    for="my-modal-4"
                    on:click={() => {
                        eventForm = currentSidebar;
                    }}
                >
                    <svg
                        xmlns="http://www.w3.org/2000/svg"
                        class="h-5 w-5"
                        fill="none"
                        viewBox="0 0 24 24"
                        stroke="currentColor"
                        ><path
                            stroke-linecap="round"
                            stroke-linejoin="round"
                            stroke-width="2"
                            d="M11 5H6a2 2 0 00-2 2v11a2 2 0 002 2h11a2 2 0 002-2v-5m-1.414-9.414a2 2 0 112.828 2.828L11.828 15H9v-2.828l8.586-8.586z"
                        /></svg
                    >
                </label>
                <button
                    class="btn btn-ghost btn-circle text-info"
                    on:click={dup}
                >
                    <svg
                        xmlns="http://www.w3.org/2000/svg"
                        class="h-5 w-5"
                        fill="none"
                        viewBox="0 0 24 24"
                        stroke="currentColor"
                        ><path
                            stroke-linecap="round"
                            stroke-linejoin="round"
                            stroke-width="2"
                            d="M8 16H6a2 2 0 01-2-2V6a2 2 0 012-2h8a2 2 0 012 2v2m-6 12h8a2 2 0 002-2v-8a2 2 0 00-2-2h-8a2 2 0 00-2 2v8a2 2 0 002 2z"
                        /></svg
                    >
                </button>
                <button
                    class="btn btn-ghost btn-circle text-error"
                    on:click={() => {
                        del();
                        // @ts-ignore
                        document.querySelector("#my-drawer-3").checked = false;
                        // currentSidebar
                    }}
                >
                    <svg
                        xmlns="http://www.w3.org/2000/svg"
                        class="h-5 w-5"
                        fill="none"
                        viewBox="0 0 24 24"
                        stroke="currentColor"
                        ><path
                            stroke-linecap="round"
                            stroke-linejoin="round"
                            stroke-width="2"
                            d="M19 7l-.867 12.142A2 2 0 0116.138 21H7.862a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16"
                        /></svg
                    >
                </button>
            </div>
        </div>

        <!-- END OF SIDEBAR CONTENT -->
    </div>
</div>

<!-- MODAL FORM -->
<input type="checkbox" id="my-modal-4" class="modal-toggle" />
<label for="my-modal-4" class="modal cursor-pointer">
    <label
        class="modal-box modal-bottom sm:modal-middle relative w-11/12 max-w-2xl"
        for=""
    >
        <!-- CONTENT HERE -->
        <div class="grid md:grid-cols-2 gap-4">
            <!-- NAME -->
            <div class="form-control w-full col-span-full">
                <label class="label" for="nunya">
                    <span class="label-text">Event name*</span>
                </label>
                <input
                    type="text"
                    bind:value={eventForm.name}
                    placeholder="Jans 50th blowout"
                    class="input input-bordered w-full"
                />
            </div>

            <!-- Start time -->
            <div class="form-control w-full">
                <label class="label" for="nunya">
                    <span class="label-text">Start time</span>
                </label>
                <input
                    type="datetime-local"
                    bind:value={eventForm.start}
                    name="time_start"
                    class="input input-bordered w-full"
                />
            </div>

            <!-- End time -->
            <div class="form-control w-full">
                <label class="label" for="nunya">
                    <span class="label-text">End time</span>
                </label>
                <input
                    type="datetime-local"
                    bind:value={eventForm.end}
                    class="input input-bordered w-full"
                />
            </div>

            <!-- Location -->
            <div class="form-control w-full col-span-full">
                <label class="label" for="nunya">
                    <span class="label-text">Location</span>
                </label>
                <input
                    type="text"
                    bind:value={eventForm.location}
                    placeholder="The open road"
                    class="input input-bordered w-full"
                />
            </div>

            <!-- Color -->
            <div class="form-control col-span-full">
                <label for="nunya" class="label">Color*</label>
                <div class="flex justify-between pr-2 pl-2">
                    <input
                        type="radio"
                        bind:group={eventForm.color}
                        name="radio-6"
                        value="primary"
                        class="radio border border-primary checked:bg-primary"
                        checked
                    />
                    <input
                        type="radio"
                        bind:group={eventForm.color}
                        name="radio-6"
                        value="secondary"
                        class="radio border border-secondary checked:bg-secondary"
                    />
                    <input
                        type="radio"
                        bind:group={eventForm.color}
                        name="radio-6"
                        value="accent"
                        class="radio border border-accent checked:bg-accent"
                    />
                    <input
                        type="radio"
                        bind:group={eventForm.color}
                        name="radio-6"
                        value="info"
                        class="radio border border-info checked:bg-info"
                    />
                    <input
                        type="radio"
                        bind:group={eventForm.color}
                        name="radio-6"
                        value="success"
                        class="radio border border-success checked:bg-success"
                    />
                    <input
                        type="radio"
                        bind:group={eventForm.color}
                        name="radio-6"
                        value="warning"
                        class="radio border border-warning checked:bg-warning"
                    />
                    <input
                        type="radio"
                        bind:group={eventForm.color}
                        name="radio-6"
                        value="error"
                        class="radio border border-error checked:bg-error"
                    />
                    <input
                        type="radio"
                        bind:group={eventForm.color}
                        name="radio-6"
                        value="ghost"
                        class="radio border border-ghost checked:bg-ghost"
                    />
                </div>
            </div>

            <div class="form-control col-span-full">
                <label class="label" for="nunya">
                    <span class="label-text">Details</span>
                </label>
                <textarea
                    bind:value={eventForm.details}
                    class="textarea textarea-bordered h-24"
                    placeholder="Do not forget the cake"
                />
            </div>

            <button
                class="btn btn-success"
                on:click={() => {
                    save(true);
                    // currentSidebar = eventForm
                }}>Save</button
            >
            <button
                class="btn btn-error btn-outline"
                on:click={() => {
                    eventForm = new EventForm();
                }}>Reset</button
            >
        </div>
    </label>
</label>
