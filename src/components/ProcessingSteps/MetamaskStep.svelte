<script>
    import { getContext } from 'svelte'

    // Components
    import MetaMaskBalance from '../ProcessingSteps/MetaMaskBalance.svelte'
    import MetaMaskTokenInput from '../ProcessingSteps/MetamaskTokenInput.svelte'
    import TokenApprovalInfo from '../ProcessingSteps/TokenApprovalInfo.svelte'

    // Misc
    import { ethStore, connected, selectedAccount, chainData } from 'svelte-web3'
    import { ethChainBalance, ethChainTokenBalance } from '../../stores/ethereumStores'
    import { getNetworkStore, swapInfo } from '../../stores/globalStores';
    import { checkChain, checkChainBalance, checkChainTokenBalance, checkChainTokenBalanceByAccount } from '../../js/ethereum-utils';   
    import { saveSwap } from '../../js/localstorage-utils'
    import { BN, eth_address_to_checksum } from '../../js/global-utils'

    export let current
    export let complete
    export let stepInfo

    let metamaskError = false
    let networkInfo = getNetworkStore(stepInfo.network)

    let clearingHouseUsdcBalances = new BN(0)

    $: hasEthBalnnce = $ethChainBalance.isGreaterThan(0)
    $: isCorrectChain = checkChain($chainData)
    $: tokenFromMe = $swapInfo.from !== "lamden"
    $: tokensToSend = $swapInfo.tokenAmount || new BN(0)
    $: hasEnoughTokens = tokensToSend.isGreaterThan(0) ? tokensToSend.isGreaterThan(0) && $ethChainTokenBalance.isGreaterThanOrEqualTo(tokensToSend) : false
    $: hasApproval = $swapInfo.metamaskApproval || false
    $: resuming = $swapInfo.burnHash || $swapInfo.depositHash || false
    $: sameAddress = resuming && $selectedAccount ? $selectedAccount.toLowerCase() === $swapInfo.metamask_address.toLowerCase() : null
    $: isLusd = $swapInfo.token.symbol === "LUSD"
    $: hasEnoughLiquidity = clearingHouseUsdcBalances.isGreaterThan(tokensToSend)

    const { nextStep, setStep, restart } = getContext('process_swap')

    chainData.subscribe(() => {
        if (complete){
            if (!checkChain()){
                if ($swapInfo.from === "lamden") setStep(1)
                else setStep(0)
            }else{
                if ($connected && $selectedAccount) refreshAllBalances()
            }
        }
    });

    selectedAccount.subscribe(refreshAllBalances)

    async function connectMetamask() {
        try {
            await ethStore.setBrowserProvider()
        } catch (error) {
            console.log(error)
            if (error.code === -32002) {
                metamaskError = 'Please open metamask and accept the connection request.'
            } else if (error.code === 4001) {
                metamaskError = 'Request Rejected.'
            } else {
                metamaskError = 'Something went wrong.'
            }
        }
    }

    function handleNextStep(){
        if (!resuming){
            swapInfo.update(curr => {
                curr.metamask_address = $selectedAccount
                return curr
            })
        }
        saveSwap()
        nextStep()
    }

    function refreshAllBalances(){
        if (!$selectedAccount) return
        checkChainBalance()
        if (isLusd) {
            const tokenAddress = $swapInfo.mintedToken.address
            const account = $networkInfo.clearingHouse.address
            checkChainTokenBalanceByAccount(tokenAddress, account).then(r => clearingHouseUsdcBalances = r)
        }

        metamaskError = false
        
        if (resuming && $selectedAccount){
            let isSameAddress = $selectedAccount.toLowerCase() === $swapInfo.metamask_address.toLowerCase()
            if (!isSameAddress) metamaskError = `Please select the same account from when the swap was created ${$swapInfo.metamask_address}`    
        }
    }

    function handleStartOver(){
        startOver()
    }

</script>


<style>
    ul{
        margin: 0;
    }

    button{
        display: block;
        margin: 1rem 0 0 auto;
    }
    .buttons > button{
        margin: 0;
        margin-top: 1rem;
        margin-left: 10px;
    }

    @media screen and (min-width: 430px) {

    }
</style>

<ul>
    <li class:yes={$connected && !metamaskError}>
        <span>
            {#if !$connected || metamaskError}
                {#if metamaskError}
                    {metamaskError}.
                {:else}
                    Not Connected
                {/if}
            {:else}
                Installed and Connected
            {/if}
        </span>
    </li>
    {#if $connected && (current || complete)}
        <li class:yes={isCorrectChain}>
            <span>
                {#if isCorrectChain}
                    Connected to {$networkInfo.networkName}
                {:else}
                    Switch Metamask to {$networkInfo.networkName}
                {/if}
            </span>
        </li>
    {/if}
    {#if $connected && isCorrectChain && (current || complete)}
        <li class:yes={hasEthBalnnce}>
            <span>
                {#if !hasEthBalnnce}
                    {`No ${$networkInfo.network_symbol} balance.`}
                {:else}
                    {`${$networkInfo.network_symbol} balance greater than zero.`}
                {/if}
            </span>
        </li>
    {/if}

    {#if isLusd && $connected && isCorrectChain && (current || complete)}
        <li class:yes={hasEnoughLiquidity}>
            <span>
                {#if !hasEnoughLiquidity}
                    {`The contract has ${clearingHouseUsdcBalances.toFixed(2)} USDC. Insufficient liquidity on ${$networkInfo.networkName}.`}
                {:else}
                    {`The contract has ${clearingHouseUsdcBalances.toFixed(2)} USDC. Liquidity is fine.`}
                {/if}
            </span>
        </li>
    {/if}

</ul>

{#if isCorrectChain && $selectedAccount}
    <MetaMaskBalance {stepInfo}/>
{/if}

{#if isCorrectChain && $connected && tokenFromMe}
    <MetaMaskTokenInput {stepInfo} input={!hasApproval} {complete}/>
{/if}

{#if $connected && hasApproval && tokenFromMe}
    <TokenApprovalInfo />
{/if}

{#if current}
    {#if $connected && $selectedAccount}
        {#if isCorrectChain}
            {#if tokenFromMe}
                <button class:success={hasEthBalnnce && hasEnoughTokens } disabled={!hasEthBalnnce || !hasEnoughTokens} on:click={handleNextStep}>
                    Next Step
                </button>
            {:else}
                {#if resuming}
                    <button class:success={hasEthBalnnce && sameAddress && (isLusd ? hasEnoughLiquidity : true)} disabled={!hasEthBalnnce || !sameAddress || (isLusd ? !hasEnoughLiquidity : false)} on:click={handleNextStep}>Resume Swap</button>
                {:else}
                    <button class:success={hasEthBalnnce && (isLusd ? hasEnoughLiquidity : true)} disabled={!hasEthBalnnce || (isLusd ? !hasEnoughLiquidity : false)} on:click={handleNextStep}>Next Step</button>
                {/if}
            {/if}
        {/if}
    {:else}
        <button on:click={connectMetamask}>Connect To MetaMask</button>
    {/if}
    <!--{`hasEthBalnnce ${hasEthBalnnce}, hasEnoughTokens ${hasEnoughTokens}`}-->
{/if}