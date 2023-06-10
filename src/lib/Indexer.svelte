<script lang="ts">
    import { getClient } from "./Client.svelte";
    import init from "@iota/sdk-wasm/web";
    import { onMount } from "svelte";
    onMount(async () => {
        try {
            await init();
        } catch (err) {
            alert(err);
        }
    });

    let pageSize = 10;
    // Limit to single page request with the empty cursor
    $: queryParameters = [{ cursor: "" }, { pageSize }];
    let outputIds = [];
    let requestingData = false;
    let output;
    let selectedOutputType = "basic";
    let usedQueryParameters = "";

    const additionalQueryParametersWithField = [
        "address",
        "tag",
        "timelockedBefore",
        "timelockedAfter",
        "createdBefore",
        "createdAfter",
    ].map((v) => {
        return { name: v, value: "" };
    });
    const additionalQueryParameters = [
        "hasNativeTokens",
        "hasTimelock",
        "hasStorageDepositReturn",
        "hasExpiration",
    ].map((v) => {
        return { name: v, value: undefined };
    });

    const getOutputIds = async () => {
        try {
            requestingData = true;
            outputIds = [];
            output = undefined;
            let client = getClient();
            let response;
            let additional = additionalQueryParameters
                .filter((qp) => qp.value != "undefined")
                .map((qp) => {
                    let obj = {};
                    obj[qp.name] = qp.value === "true";
                    return obj;
                });
            for (let qp of additionalQueryParametersWithField) {
                console.log(qp);
                if (qp.value != "") {
                    let obj = {};
                    // Parse integer values
                    if (
                        qp.name.startsWith("time") ||
                        qp.name.startsWith("created")
                    ) {
                        obj[qp.name] = parseInt(qp.value);
                    } else if (qp.name == "tag") {
                        if (qp.value.startsWith("0x")) {
                            obj[qp.name] = qp.value;
                        } else {
                            obj[qp.name] = utf8ToHex(qp.value);
                        }
                    } else {
                        obj[qp.name] = qp.value;
                    }
                    additional.push(obj);
                }
            }
            let finalQueryParameters = additional.concat(queryParameters);
            usedQueryParameters = JSON.stringify(finalQueryParameters);
            switch (selectedOutputType) {
                case "alias":
                    response = await client.aliasOutputIds(
                        finalQueryParameters
                    );
                    break;
                case "basic":
                    response = await client.basicOutputIds(
                        finalQueryParameters
                    );
                    break;
                case "foundry":
                    response = await client.foundryOutputIds(
                        finalQueryParameters
                    );
                    break;
                case "nft":
                    response = await client.nftOutputIds(finalQueryParameters);
                    break;
            }

            console.log(response);
            outputIds = outputIds.concat(response.items);
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

    const utf8ToHex = (str: string) => {
        let result = "0x";
        for (let i = 0; i < str.length; i++) {
            let hex = str.charCodeAt(i).toString(16);
            result += hex.slice(-4);
        }
        return result;
    };
</script>

<main>
    Page size: <input
        type="number"
        bind:value={pageSize}
        placeholder="page size"
    />
    <div class="outputTypeSelector">
        Output type:
        {#each ["alias", "basic", "foundry", "nft"] as type}
            <!-- Highlight selected type -->
            {#if type == selectedOutputType}
                <button
                    class="selected"
                    on:click={() => (selectedOutputType = type)}>{type}</button
                >
            {:else}
                <button on:click={() => (selectedOutputType = type)}
                    >{type}</button
                >
            {/if}
        {/each}
    </div>
    <div style="display: grid; grid-template-columns: 1fr 1fr;">
        <div>
            {#each additionalQueryParametersWithField as qp}
                <br />
                {qp.name}:
                <input type="string" size="60" bind:value={qp.value} />
            {/each}
        </div>
        <div>
            <ul style="list-style-type:none; float: left;">
                {#each additionalQueryParameters as qp, index}
                    <li>
                        {qp.name}
                        <select bind:value={qp.value}>
                            <option value="undefined">undefined</option>
                            <option value="false">false</option>
                            <option value="true">true</option>
                        </select>
                    </li>
                {/each}
            </ul>
        </div>
    </div>
    <button on:click={getOutputIds}>get output ids</button>
    <br />
    Used query parameters: {usedQueryParameters}
    <br />
    {#if requestingData}
        <div style="color: red;">Requesting data...</div>
        <br />
    {/if}

    <div style="display: grid; grid-template-columns: 1fr 1fr;">
        <!-- Only show the table if there are any created outputs -->
        {#if outputIds.length > 0}
            <table>
                <tr>
                    <th>Output id</th>
                </tr>
                {#each [...outputIds].reverse() as outputId}
                    <tr>
                        <td>
                            <div style="display: list; font-family: monospace;">
                                <div>
                                    <button
                                        on:click={() => getOutput(outputId)}
                                    >
                                        {outputId}</button
                                    >
                                </div>
                            </div>
                        </td>
                    </tr>
                {/each}
            </table>
            {#if output}
                <pre style="text-align: left;">
                  {JSON.stringify(output, null, 2)}
                </pre>
                <br />
            {/if}
        {/if}
    </div>
</main>

<style>
    .outputTypeSelector {
        display: table-row;
    }
    .selected {
        background-color: rgb(160, 160, 160);
    }
    button {
        margin: 2px;
    }
</style>
