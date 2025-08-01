<div class="content">
  <div class="col">
    <Card footer footerRight ref="filter-card">
      <span slot="title">
        <i class="fas fa-filter"></i>
        Filter Logs
      </span>

      <div slot="body" class="body-wrapper">
        <div class="form-wrapper">
          <div class="row">
            <Input col4=true label="Ticket ID" placeholder="Ticket ID"
                   on:input={handleInputTicketId} bind:value={filterSettings.ticketId}/>

            <Input col4=true label="Username" placeholder="Username" on:input={handleInputUsername}
                   bind:value={filterSettings.username}/>

            <Input col4=true label="User ID" placeholder="User ID" on:input={handleInputUserId}
                   bind:value={filterSettings.userId}/>

            <Input col4=true label="Closed By Id" placeholder="Closed By" on:input={handleInputClosedById}
                  bind:value={filterSettings.closedById}/>
          </div>
          <div class="row">
            <div class="col-4">
              <PanelDropdown label="Panel" isMulti={false} bind:panels bind:selected={selectedPanel} />
            </div>

            <Dropdown col4=true label="Rating" bind:value={filterSettings.rating}>
              <option value=0>Any</option>
              <option value=1>1 ⭐</option>
              <option value=2>2 ⭐</option>
              <option value=3>3 ⭐</option>
              <option value=4>4 ⭐</option>
              <option value=5>5 ⭐</option>
            </Dropdown>

            <Input col4=true label="Claimed By Id" placeholder="Claimed By" on:input={handleInputClaimedById}
                  bind:value={filterSettings.claimedById}/>
          </div>
        </div>
      </div>
      <div slot="footer">
        <Button icon="fas fa-search" on:click={filter}>Filter</Button>
      </div>
    </Card>

    <div style="margin: 2% 0;">
      <Card footer="{false}">
        <span slot="title">
          Transcripts
        </span>
        <ColumnSelector
        options={["Ticket ID", "Username", "Rating", "Close Reason", "Transcript"]}
        bind:selected={selectedColumns}
        slot="title-items"
        />

        <div slot="body" class="main-col">
          <table class="nice">
            <thead>
            <tr>
              <th class:visible={selectedColumns.includes('Ticket ID')}>Ticket ID</th>
              <th class:visible={selectedColumns.includes('Username')}>Username</th>
              <th class:visible={selectedColumns.includes('Rating')}>Rating</th>
              <th class:visible={selectedColumns.includes('Close Reason')}>Close Reason</th>
              <th class:visible={selectedColumns.includes('Transcript')}>Transcript</th>
            </tr>
            </thead>
            <tbody>
            {#each transcripts as transcript}
              <tr>
                <td class:visible={selectedColumns.includes('Ticket ID')}>{transcript.ticket_id}</td>
                <td class:visible={selectedColumns.includes('Username')}>{transcript.username}</td>
                <td class:visible={selectedColumns.includes('Rating')}>
                  {#if transcript.rating}
                    {transcript.rating} ⭐
                  {:else}
                    No rating
                  {/if}
                </td>
                <td class:visible={selectedColumns.includes('Close Reason')}>
                  {transcript.close_reason || 'No reason specified'}
                </td>
                <td class:visible={selectedColumns.includes('Transcript')}>
                  {#if transcript.has_transcript}
                    <Navigate to="{`/manage/${guildId}/transcripts/view/${transcript.ticket_id}`}" styles="link">
                      <Button>View</Button>
                    </Navigate>
                  {/if}
                </td>
              </tr>
            {/each}
            </tbody>
          </table>

          <div class="nav" class:nav-margin={transcripts.length === 0}>
            <i class="fas fa-chevron-left" class:hidden={page === 1} on:click={loadPrevious}></i>
            <span>Page {page}</span>
            <i class="fas fa-chevron-right"
               class:hidden={transcripts.length < pageLimit || transcripts[transcripts.length - 1].ticket_id === 1}
               on:click={loadNext}></i>
          </div>
        </div>
      </Card>
    </div>
  </div>
</div>

<script>
    import Card from '../components/Card.svelte'
    import Input from '../components/form/Input.svelte'
    import Button from '../components/Button.svelte'
    import {notifyError, withLoadingScreen} from '../js/util'
    import {onMount} from "svelte";
    import {dropdown} from "../js/stores";
    import axios from "axios";
    import {API_URL} from "../js/constants";
    import {setDefaultHeaders} from '../includes/Auth.svelte'
    import {Navigate} from 'svelte-router-spa'
    import PanelDropdown from "../components/PanelDropdown.svelte";
    import Dropdown from "../components/form/Dropdown.svelte";
    import ColumnSelector from "../components/ColumnSelector.svelte";

    setDefaultHeaders();

    export let currentRoute;
    let guildId = currentRoute.namedParams.id

    let filterSettings = {};
    let transcripts = [];

    let panels = [];
    let selectedPanel;

    const pageLimit = 15;
    let page = 1;

    // Show Columns logic
    let selectedColumns = ['Ticket ID', 'Username', 'Rating', 'Close Reason', 'Transcript'];
    const columnStorageKey = 'transcript_list:selected_columns';

    $: selectedColumns, updateColumnStorage();

    function updateColumnStorage() {
      window.localStorage.setItem(columnStorageKey, JSON.stringify(selectedColumns));
    }

    function loadColumnSettings() {
      const columns = window.localStorage.getItem(columnStorageKey);
      if (columns) {
        selectedColumns = JSON.parse(columns);
      }
    }

    let handleInputTicketId = () => {
        filterSettings.username = undefined;
        filterSettings.userId = undefined;

        if (filterSettings.ticketId === "") {
            filterSettings.ticketId = undefined;
        }
    };

    let handleInputUsername = () => {
        filterSettings.ticketId = undefined;
        filterSettings.userId = undefined;

        if (filterSettings.username === "") {
            filterSettings.username = undefined;
        }
    };

    let handleInputUserId = () => {
        filterSettings.ticketId = undefined;
        filterSettings.username = undefined;

        if (filterSettings.userId === "") {
           filterSettings.userId = undefined;
        }
    };

    let handleInputClosedById = () => {
        if (filterSettings.closedById == "") {
            filterSettings.closedById = undefined;
        }
    };

    let handleInputClaimedById = () => {
        if (filterSettings.claimedById == "") {
            filterSettings.claimedById = undefined;
        }
    };

    let loading = false;

    async function loadPrevious() {
        if (loading) return;

        if (page === 1) {
            return;
        }

        let paginationSettings = buildPaginationSettings(page - 1);

        loading = true;
        if (await loadData(paginationSettings)) {
            page--;
        }
        loading = false;
    }

    async function loadNext() {
        if (loading) return;

        if (transcripts.length < pageLimit || transcripts[transcripts.length - 1].ticket_id === 1) {
            return;
        }

        let paginationSettings = buildPaginationSettings(page + 1);

        loading = true;
        if (await loadData(paginationSettings)) {
            page++;
        }
        loading = false;
    }

    function buildPaginationSettings(page) {
        // Undefined fields won't be included in the JSON
        return {
            id: filterSettings.ticketId,
            username: filterSettings.username,
            user_id: filterSettings.userId,
            closed_by_id: filterSettings.closedById,
            claimed_by_id: filterSettings.claimedById,
            rating: filterSettings.rating,
            panel_id: selectedPanel,
            page: page,
        };
    }

    async function filter() {
        let opts = buildPaginationSettings(1);
        await loadData(opts);
        page = 1;
    }

    async function loadPanels() {
        const res = await axios.get(`${API_URL}/api/${guildId}/panels`);
        if (res.status !== 200) {
            notifyError(res.data.error);
            return;
        }

        panels = res.data;
    }

    async function loadData(paginationSettings) {
        const res = await axios.post(`${API_URL}/api/${guildId}/transcripts`, paginationSettings);
        if (res.status !== 200) {
            notifyError(res.data.error);
            return false;
        }

        transcripts = res.data;
        return true;
    }

    withLoadingScreen(async () => {
        loadColumnSettings();
        await Promise.all([
            loadPanels(),
            loadData({})
        ])
    })
</script>

<style>
    .content {
        display: flex;
        justify-content: center;
        height: 100%;
        width: 100%;
    }

    .col {
        display: flex;
        flex-direction: column;
        height: 100%;
        width: 100%;
    }

    .main-col {
        display: flex;
        flex-direction: column;
        width: 100%;
        height: 100%;
    }

    .row {
        display: flex;
        flex-direction: row;
        justify-content: flex-start;
        gap: 2%;
        width: 100%;
        height: 100%;
    }

    .form-wrapper {
        display: flex;
        flex-direction: column;
        width: 100%;
        height: 100%;
    }

    :global([ref=filter-card]) {
        min-height: 110px !important;
    }

    :global(table.nice) {
        width: 100%;
        border-collapse: collapse;
    }

    :global(table.nice > thead > tr > th) {
        text-align: left;
        font-weight: normal;
        border-bottom: 1px solid #dee2e6;
        padding-left: 10px;
        padding-right: 10px;
    }

    :global(table.nice > thead > tr, table.nice > tbody > tr) {
        border-bottom: 1px solid #dee2e6;
    }

    :global(table.nice > tbody > tr:last-child) {
        border-bottom: none;
    }

    :global(table.nice > tbody > tr > td) {
        padding: 10px 0 10px 10px;
    }

    .nav {
        display: flex;
        flex-direction: row;
        justify-content: center;
        align-items: center;
    }

    .nav > i {
        color: #1dc7ea;
        cursor: pointer;
    }

    .nav > span {
        margin: 0 5px;
    }

    .nav-margin {
        margin-top: 15px;
    }

    .hidden {
        color: #6c757d !important;
        cursor: default !important;
    }

    @media only screen and (max-width: 950px) {
        .row {
            flex-direction: column;
        }

        :global([ref=filter-card]) {
            min-height: 252px !important;
        }
    }

    @media only screen and (max-width: 576px) {
        .col {
            width: 100%;
        }
    }

    th, td {
        display: none;
    }
    th.visible, td.visible {
        display: table-cell;
    }
</style>
