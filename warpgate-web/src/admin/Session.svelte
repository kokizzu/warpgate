<script lang="ts">
import { api, SessionSnapshot, Recording, TargetSSHOptions, TargetHTTPOptions } from 'admin/lib/api'
import { timeAgo } from 'admin/lib/time'
import AsyncButton from 'common/AsyncButton.svelte'
import moment from 'moment'
import { onDestroy } from 'svelte'
import { link } from 'svelte-spa-router'
import { Alert, Spinner } from 'sveltestrap'
import LogViewer from './LogViewer.svelte'
import RelativeDate from './RelativeDate.svelte'

export let params = { id: '' }

let error: Error|null = null
let session: SessionSnapshot|null = null
let recordings: Recording[]|null = null

async function load () {
    session = await api.getSession(params)
    recordings = await api.getSessionRecordings(params)
}

async function close () {
    api.closeSession(session!)
}

function getTargetDescription () {
    if (session?.target) {
        let address = '<unknown>'
        if (session.target.options.kind === 'TargetSSHOptions') {
            const options = session.target.options as TargetSSHOptions
            address = `${options.host}:${options?.port}`
        }
        if (session.target.options.kind === 'TargetHTTPOptions') {
            const options = session.target.options as unknown as TargetHTTPOptions
            address = options.url
        }
        return `${session.target.name} (${address})`
    } else {
        return 'Not selected yet'
    }
}

load().catch(e => {
    error = e
})

const interval = setInterval(load, 1000)
onDestroy(() => clearInterval(interval))

</script>

{#if !session && !error}
    <Spinner />
{/if}

{#if error}
    <Alert color="danger">{error}</Alert>
{/if}

{#if session}
    <div class="page-summary-bar">
        <div>
            <h1>Session</h1>
            <div>
                <strong class="me-2">
                    {#if session.username}
                        {session.username}
                    {:else}
                        Logging in
                    {/if}
                    ⇆
                    {getTargetDescription()}
                </strong>
                <span class="text-muted">
                    {#if session.ended}
                        {moment.duration(moment(session.ended).diff(session.started)).humanize()} long, <RelativeDate date={session.started} />
                    {:else}
                        {moment.duration(moment().diff(session.started)).humanize()}
                    {/if}
                </span>
            </div>
        </div>
        {#if !session.ended}
            <div class="ms-auto">
                <AsyncButton outline click={close}>
                    Close now
                </AsyncButton>
            </div>
        {/if}
    </div>

    {#if recordings?.length }
        <h3 class="mt-4">Recordings</h3>
        <div class="list-group list-group-flush">
            {#each recordings as recording}
                <a
                    class="list-group-item list-group-item-action"
                    href="/recordings/{recording.id}"
                    use:link>
                    <div class="main">
                        <strong>
                            {recording.name}
                        </strong>
                        <small class="meta ms-auto">
                            {timeAgo(recording.started)}
                        </small>
                    </div>
                </a>
            {/each}
        </div>
    {/if}

    <h3 class="mt-4">Log</h3>
    <LogViewer filters={{
        sessionId: session.id,
    }} />

{/if}

<style lang="scss">
.list-group-item {
    .main {
        display: flex;
        align-items: center;

        > * {
            margin-right: 20px;
        }
    }

    .meta {
        opacity: .75;
    }
}
</style>