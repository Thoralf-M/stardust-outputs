<script lang="ts">
    import { getClient } from "./Client.svelte";
    import init from "@iota/sdk-wasm/web";
    import { onMount } from "svelte";
    onMount(async () => {
        try {
            await init();
            await updateLatestMilestoneIndex();
        } catch (err) {
            alert(err);
        }
    });
    let startIndex = 0;
    let endIndex = 0;
    let utxoChanges = [];
    let requestingData = false;
    let output;

    const getUtxoChanges = async () => {
        try {
            requestingData = true;
            utxoChanges = [];
            output = undefined;
            let client = getClient();
            for (let index = startIndex; index <= endIndex; index++) {
                utxoChanges = utxoChanges.concat(
                    await client.getUtxoChangesByIndex(index)
                );
            }
            console.log(utxoChanges);
            requestingData = false;
        } catch (err) {
            requestingData = false;
            alert(err);
        }
    };

    const getOutput = async (outputId) => {
        try {
            let client = getClient();
            output = await client.getOutput(outputId);
        } catch (err) {
            alert(err);
        }
    };

    export const updateLatestMilestoneIndex = async () => {
        try {
            let nodeInfo = await getClient().getInfo();
            endIndex = nodeInfo.nodeInfo.status.latestMilestone.index;
            if (endIndex > 8) {
                startIndex = endIndex - 9;
            }
        } catch (err) {
            alert(err);
        }
    };
</script>

<main>
    <input
        type="number"
        size="10"
        bind:value={startIndex}
        placeholder="milestone start index"
    />
    <input
        type="number"
        size="10"
        bind:value={endIndex}
        placeholder="milestone start index"
    />
    <button on:click={updateLatestMilestoneIndex}>
        range latest 10 milestones</button
    >
    <br />
    <button on:click={getUtxoChanges}>get utxo changes</button>
    <br />
    {#if requestingData}
        <div style="color: red;">
            Requesting data...
            <!-- TODO: https://svelte.dev/examples/tweened -->
        </div>
        <br />
    {/if}

    <div style="display: grid; grid-template-columns: 1fr 1fr;">
        <!-- Only show the table if there are any created outputs -->
        {#if utxoChanges.length > 0 && utxoChanges.some((e) => e.createdOutputs.length > 0)}
            <table>
                <tr>
                    <th>Milestone index</th>
                    <th>Created outputs</th>
                </tr>
                {#each [...utxoChanges].reverse() as milestoneChange}
                    {#if milestoneChange.createdOutputs.length > 0}
                        <tr>
                            <td>
                                {milestoneChange.index}
                            </td>
                            <td>
                                <div
                                    style="display: list; font-family: monospace;"
                                >
                                    {#each milestoneChange.createdOutputs as created}
                                        <div>
                                            <button
                                                on:click={() =>
                                                    getOutput(created)}
                                            >
                                                {created}</button
                                            >
                                        </div>
                                    {/each}
                                </div>
                            </td>
                        </tr>
                    {/if}
                {/each}
            </table>
            {#if output}
                <pre style="text-align: left;">
                  {JSON.stringify(output, null, 2)}
                </pre>
                <br />
            {/if}
        {:else}
            No new outputs
        {/if}
    </div>
</main>
