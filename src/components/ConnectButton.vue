<script>
import MetamaskLogo from 'src/assets/metamask-fox.svg'
import { mapGetters, mapMutations, mapState } from 'vuex';
import { ethers } from 'ethers';
import { WEI_PRECISION } from 'src/lib/utils';
const providersError = 'More than one provider is active, disable additional providers.';
const unsupportedError ='current EVM wallet provider is not supported.';
const LOGIN_EVM = 'evm';
const LOGIN_NATIVE = 'native';
const PROVIDER_WEB3_INJECTED = 'injectedWeb3'

export const triggerLogin = () => document.querySelector('#c-connect-button__login-button')?.click();

export default {
    name: 'ConnectButton',
    data() {
        return {
            tab: 'web3',
            showLogin: false,
            metamaskLogo: MetamaskLogo,
        }
    },
    computed: {
        ...mapGetters('login', [
            'isLoggedIn',
            'isNative',
            'address',
            'nativeAccount',
        ]),
        ...mapState('general', ['browserSupportsEthereum']),
    },
    async mounted() {
        const loginData = localStorage.getItem('loginData');
        if (!loginData)
            return;

        const loginObj = JSON.parse(loginData);
        if (loginObj.type === LOGIN_EVM) {
            const provider = this.getInjectedProvider();
            let checkProvider = new ethers.providers.Web3Provider(provider)
            const {chainId} = await checkProvider.getNetwork();
            if(loginObj.chain == chainId){
                switch (loginObj.provider) {
                case PROVIDER_WEB3_INJECTED:
                    this.injectedWeb3Login();
                    break;
                default:
                    console.error(`Unknown web3 login type: ${loginObj.provider}`);
                    break;
                }
            }
        } else if (loginObj.type === LOGIN_NATIVE) {
            const wallet = this.$ual.authenticators.find(a => a.getName() == loginObj.provider);
            this.ualLogin(wallet);
        }
    },
    methods: {
        ...mapMutations('login', [
            'setLogin',
        ]),
        getLoginDisplay() {
            return this.isNative ? this.nativeAccount : `0x...${this.address.slice(this.address.length - 4)}`;
        },
        connect() {
            this.showLogin = true;
        },
        disconnect() {
            if (this.isNative) {
                const loginData = localStorage.getItem('loginData');
                if (!loginData)
                    return;

                const loginObj = JSON.parse(loginData);
                const wallet = this.$ual.authenticators.find(a => a.getName() == loginObj.provider);
                wallet.logout();
            }

            this.setLogin({});
            localStorage.removeItem('loginData');
            this.$providerManager.setProvider(null);
        },
        goToAddress() {
            this.$router.push(`/address/${this.address}`);
        },
        async injectedWeb3Login() {
            if (!this.browserSupportsEthereum) {
                window.open('https://metamask.app.link/dapp/teloscan.io');
                return;
            }

            const address = await this.getInjectedAddress();
            if (address) {
                this.setLogin({
                    address,
                })
                let provider = this.getInjectedProvider();
                let checkProvider = new ethers.providers.Web3Provider(provider)
                this.$providerManager.setProvider(provider);
                const {chainId} = await checkProvider.getNetwork();
                localStorage.setItem('loginData', JSON.stringify({type: LOGIN_EVM, provider: PROVIDER_WEB3_INJECTED, chain: chainId }));
                provider.on('chainChanged', (newNetwork) => {
                    if(newNetwork != chainId){
                        this.setLogin({});
                        this.$providerManager.setProvider(null);
                    }
                });
                provider.on('accountsChanged', (accounts) => {
                    this.setLogin({
                        address: accounts[0],
                    })
                })
            }
            this.showLogin = false;
        },
        async ualLogin(wallet, account) {
            await wallet.init();
            const users = await wallet.login(account);
            if (users.length) {
                const account = users[0];
                const accountName = await account.getAccountName();
                let evmAccount;
                try {
                    evmAccount = await this.$evm.telos.getEthAccountByTelosAccount(accountName);
                } catch (e) {
                    this.$q.notify({
                        position: 'top',
                        message: `Search for EVM address linked to ${accountName} native account failed.  You can create one at wallet.telos.net`,
                        timeout: 6000,
                    });
                    wallet.logout();
                    return;
                }
                this.setLogin({
                    address: evmAccount.address,
                    nativeAccount: accountName,
                })
                this.$providerManager.setProvider(account);
                localStorage.setItem('loginData', JSON.stringify({type: LOGIN_NATIVE, provider: wallet.getName()}));
            }
            this.showLogin = false;
        },
        async getInjectedAddress() {
            const provider = this.getInjectedProvider();
            let checkProvider = new ethers.providers.Web3Provider(provider);

            const newProviderInstance = await this.ensureCorrectChain(checkProvider);
            if(newProviderInstance){
                checkProvider = newProviderInstance;
            }
            const accounts = await checkProvider.listAccounts();
            if (accounts.length > 0) {
                checkProvider = await this.ensureCorrectChain(checkProvider);
                return accounts[0];
            } else {
                const accessGranted = await provider.request({ method: 'eth_requestAccounts' });

                if (accessGranted.length < 1) {
                    return false;
                }

                checkProvider = await this.ensureCorrectChain(checkProvider);
                return accessGranted[0];
            }
        },
        async ensureCorrectChain(checkProvider) {
            const {chainId} = await checkProvider.getNetwork();
            if (chainId !== process.env.NETWORK_EVM_CHAIN_ID) {
                await this.switchChainInjected();
                const provider = this.getInjectedProvider();
                return new ethers.providers.Web3Provider(provider);
            }
        },
        getInjectedProvider() {
            const provider = window.ethereum.isMetaMask || window.ethereum.isCoinbaseWallet ?
                window.ethereum :
                null
            if (!provider) {
                console.error(providersError, 'or', unsupportedError);
            }
            return provider;
        },
        async switchChainInjected() {
            const provider = this.getInjectedProvider();

            if (provider) {
                const chainId = parseInt(process.env.NETWORK_EVM_CHAIN_ID, 10);
                const chainIdParam = `0x${chainId.toString(16)}`;
                const mainnet = chainId === 40;
                try {
                    await provider.request({
                        method: 'wallet_switchEthereumChain',
                        params: [{chainId: chainIdParam}],
                    });
                    return true;
                } catch (e) {
                    const chainNotAddedCodes = [
                        4902,
                        -32603, // https://github.com/MetaMask/metamask-mobile/issues/2944
                    ];

                    if (chainNotAddedCodes.includes(e.code)) {  // 'Chain <hex chain id> hasn't been added'
                        try {
                            await provider.request({
                                method: 'wallet_addEthereumChain',
                                params: [{
                                    chainId: chainIdParam,
                                    chainName: `Telos EVM ${mainnet ? 'Mainnet' : 'Testnet'}`,
                                    nativeCurrency: {
                                        name: 'Telos',
                                        symbol: 'TLOS',
                                        decimals: WEI_PRECISION,
                                    },
                                    rpcUrls: [`https://${mainnet ? 'mainnet' : 'testnet'}.telos.net/evm`],
                                    blockExplorerUrls: [`https://${mainnet ? '' : 'testnet.'}teloscan.io`],
                                }],
                            });
                            return true;
                        } catch (e) {
                            console.error(e);
                        }
                    }
                }
            } else {
                return false;
            }
        },
    },
}
</script>
<template>
<div class="c-connect-button">
    <q-btn
        v-if="!isLoggedIn"
        id="c-connect-button__login-button"
        label="Connect Wallet"
        @click="connect"
    />

    <q-btn-dropdown flat
                    round
                    v-else
                    :label="getLoginDisplay()">
        <q-list>
            <q-item clickable="clickable" v-close-popup @click="goToAddress()">
                <q-item-section>
                    <q-item-label>View address</q-item-label>
                </q-item-section>
            </q-item>
            <q-item clickable="clickable" v-close-popup @click="disconnect()">
                <q-item-section>
                    <q-item-label>Disconnect</q-item-label>
                </q-item-section>
            </q-item>
        </q-list>
    </q-btn-dropdown>

    <q-dialog v-model="showLogin">
        <q-card rounded="rounded">
            <q-tabs v-model="tab">
                <q-tab name="web3" label="EVM Wallets"></q-tab>
                <q-tab name="native" label="Native Wallets"></q-tab>
            </q-tabs>
            <q-separator/>
            <q-tab-panels v-model="tab" animated="animated">
                <q-tab-panel name="web3">
                    <q-card class="wallet-icon cursor-pointer" @click="injectedWeb3Login()">
                        <q-img class="wallet-img" :src="metamaskLogo"></q-img>
                        <p>{{ !browserSupportsEthereum ? 'Continue on ' : '' }}Metamask</p>
                    </q-card>
                </q-tab-panel>
                <q-tab-panel name="native">
                    <q-card
                        class="wallet-icon cursor-pointer"
                        v-for="wallet in $ual.authenticators"
                        :key="wallet.getStyle().text"
                        @click="ualLogin(wallet)"
                    >
                        <q-img class="wallet-img" :src="wallet.getStyle().icon"></q-img>
                        <p>{{ wallet.getStyle().text }}</p>
                    </q-card>
                </q-tab-panel>
            </q-tab-panels>
        </q-card>
    </q-dialog>
</div>
</template>

<style lang='sass'>
    .wallet-icon
        width: 42%
        display: inline-block
        margin: .5rem
        padding: .5rem
        text-align: center
        p
            margin: .25rem

    .wallet-img
        width: 3.5rem
        margin: .5rem .5rem 0 .5rem

    .q-menu
        margin-top: 10px !important

@media only screen and (max-width: 550px)
    .wallet-icon
        width: 92%
    .connect-button
        margin-left: 5px
    .q-btn
        font-size: 0.9em
        padding: 4px 10px

</style>
