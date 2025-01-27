<template>
<div>
    <q-dialog v-model="enterAmount">
        <q-card class="amount-dialog">
            <p>Select number of decimals and enter an amount, this will be entered for you into the function parameter as uint256</p>
            <q-select
                v-model="selectDecimals"
                :options="decimalOptions"
                @input="updateDecimals"
            />
            <q-input
                v-if="selectDecimals.value === 'custom'"
                v-model.number="customDecimals"
                type="number"
                label="Custom decimals"
                @change="updateDecimals"
            />
            <q-input
                v-model="amountInput"
                label="Amount"
                type="number"
            />
            <q-card-actions align="right">
                <q-btn
                    v-close-popup
                    flat="flat"
                    label="Ok"
                    color="primary"
                    @click="setAmount"
                />
                <q-btn
                    v-close-popup
                    flat="flat"
                    label="Cancel"
                    color="primary"
                    @click="clearAmount"
                />
            </q-card-actions>
        </q-card>
    </q-dialog>
    <div v-if="abi.stateMutability === 'payable'">
        <q-input
            v-model="value"
            label="Value (amount)"
        >
            <template #append>
                <q-icon
                    class="cursor-pointer"
                    name="pin"
                    @click="showAmountDialog('value')"
                />
            </template>
        </q-input>
    </div>
    <div v-for="(param, idx) in abi.inputs" :key="idx">
        <q-input
            v-model="params[idx]"
            :label="makeLabel(param, idx)"
            :placeholder="getHintForInput(param.type)"
        >
            <template v-if="parameterTypeIsUint256(param.type)" #append>
                <q-icon
                    class="cursor-pointer"
                    name="pin"
                    @click="showAmountDialog(idx)"
                />
            </template>
        </q-input>
        <p v-if="!parameterTypeIsImplemented(param.type)" class="q-px-md">
            ⚠️Warning: Implementation of input for type <code>{{ param.type }}</code> is under development; argument
            will be passed as the exact string entered. This may lead to unexpected results if the method is not
            expecting a string.
        </p>

    </div>
    <q-btn
        v-if="enableRun"
        :loading="loading"
        :label="runLabel"
        :disabled="missingInputs"
        class="run-button"
        color="primary"
        icon="send"
        @click="run"
    />
    <p class="text-red output-container">
        {{ errorMessage }}
    </p>
    <div v-if="result" class="output-container">
        Result ({{ abi.outputs && abi.outputs.length > 0 ? abi.outputs[0].type : '' }}): {{ result }}
    </div>
    <div
        v-if="hash"
        class="output-container"
    >
        View Transaction:&nbsp;
        <transaction-field :transaction-hash="hash" />
    </div>
</div>
</template>

<script>
import { mapGetters } from 'vuex';
import { BigNumber, ethers } from 'ethers';
import { Transaction } from '@ethereumjs/tx';

import {
    getExpectedArrayLengthFromParameterType,
    parameterTypeIsImplemented,
    parameterTypeIsUint256,
    parameterTypeIsAddress,
    parameterTypeIsUint256Array,
    parameterTypeIsAddressArray,
    parameterTypeIsBoolean,
    parameterTypeIsBooleanArray,
    parseUint256String,
    parseUint256ArrayString,
    parseAddressString,
    parseAddressArrayString,
    parseBooleanString,
    parseBooleanArrayString,
    parameterTypeIsString,
} from 'components/ContractTab/function-interface-utils';

import TransactionField from 'components/TransactionField';

const decimalOptions = [{
    label: '18 - TLOS/ETH/etc..',
    value: 18,
}, {
    label: '9 - Gwei',
    value: 9,
}, {
    label: '8 - BTC',
    value: 8,
}, {
    label: '0 - Wei',
    value: 0,
}, {
    label: 'Custom',
    value: 'custom',
}];

export default {
    name: 'FunctionInterface',
    components: {
        TransactionField,
    },
    props: {
        contract: {
            type: Object,
            default: null,
        },
        abi: {
            type: Object,
            default: null,
        },
        runLabel: {
            type: String,
            default: null,
        },
    },
    data : () => ({
        loading: false,
        errorMessage: '',
        decimalOptions,
        result: null,
        hash: null,
        enterAmount: false,
        amountInput: 0,
        amountParam: null,
        amountDecimals: 0,
        selectDecimals: decimalOptions[0],
        customDecimals: 0,
        value: '0',
        params: [],
        valueParam: {
            'name': 'value',
            'type': 'amount',
            'internalType': 'amount',
        },
    }),
    computed: {
        ...mapGetters('login', [
            'address',
            'isLoggedIn',
            'isNative',
            'nativeAccount',
        ]),
        enableRun() {
            return this.isLoggedIn || this.abi.stateMutability === 'view'
        },
        missingInputs() {
            if (this.abi.inputs.length !== this.params.length) {
                return true;
            }

            for (let i = 0; i < this.abi.inputs.length; i++) {
                if (!this.params[i]) {
                    return true;
                }
            }

            return false;
        },
    },
    methods: {
        parameterTypeIsImplemented,
        parameterTypeIsUint256,
        makeLabel(abiParam, position) {
            return `${abiParam.name ? abiParam.name : `Param ${position}`} (${abiParam.type})`
        },
        showAmountDialog(param) {
            this.amountParam = param;
            this.amountDecimals = 18;
            this.enterAmount = true;
        },
        updateDecimals() {
            this.amountDecimals = this.selectDecimals.value === 'custom' ? this.customDecimals : this.selectDecimals.value;
        },
        setAmount() {
            const integerAmount = ethers.utils.parseUnits(this.amountInput + '', this.amountDecimals).toString();
            if (this.amountParam === 'value')
                this.value = integerAmount;
            else
                this.params[this.amountParam] = integerAmount;

            this.clearAmount();
        },
        clearAmount() {
            this.amountInput = 0;
        },
        getFormattedParams() {
            const formatted = [];
            for (let i = 0; i < this.abi.inputs.length; i++) {
                let param = this.abi.inputs[i];
                formatted.push(this.formatValue(this.params[i], param.type));
            }

            return formatted;
        },
        getHintForInput(type) {
            let example;

            if (parameterTypeIsAddress(type)) {
                example = '0x0000000000000000000000000000000000000000';
            } else if (parameterTypeIsAddressArray(type)) {
                example = '[0x0000000000000000000000000000000000000000, 0x1111111111111111111111111111111111111111]';
            } else if (parameterTypeIsUint256(type)) {
                example = '12345';
            } else if (parameterTypeIsUint256Array(type)) {
                example = '[1234, 5678]';
            } else if (parameterTypeIsBoolean(type)) {
                example = 'false';
            } else if (parameterTypeIsBooleanArray(type)) {
                example = '[false, true]';
            } else if (parameterTypeIsString(type)) {
                example = 'Example string';
            }

            if (example)
                return `e.g. ${example}`;

            return '';
        },
        formatValue(rawValue, type) {
            const value = rawValue.trim();
            const expectedArrayLength = getExpectedArrayLengthFromParameterType(type);

            const typeIsUint256      = parameterTypeIsUint256(type);
            const typeIsAddress      = parameterTypeIsAddress(type);
            const typeIsUint256Array = parameterTypeIsUint256Array(type);
            const typeIsAddressArray = parameterTypeIsAddressArray(type);
            const typeIsBoolean      = parameterTypeIsBoolean(type);
            const typeIsBooleanArray = parameterTypeIsBooleanArray(type);
            const typeIsString       = parameterTypeIsString(type);

            let parsedValue;

            if (typeIsUint256) {
                parsedValue = parseUint256String(value);
            } else if (typeIsUint256Array) {
                parsedValue = parseUint256ArrayString(value, expectedArrayLength);
            } else if (typeIsAddress) {
                parsedValue = parseAddressString(value);
            } else if (typeIsAddressArray) {
                parsedValue = parseAddressArrayString(value, expectedArrayLength);
            } else if (typeIsBoolean) {
                parsedValue = parseBooleanString(value);
            }  else if (typeIsBooleanArray) {
                parsedValue = parseBooleanArrayString(value, expectedArrayLength);
            } else if (typeIsString) {
                return value;
            } else {
                return value;
            }

            if (parsedValue === undefined) {
                throw `Invalid value for type ${type}`;
            }

            return parsedValue;
        },
        async run() {
            this.loading = true;

            try {
                const opts = {};
                if (this.abi.stateMutability === 'payable') {
                    opts.value = this.formatValue(this.value, 'uint256');
                }

                if (this.abi.stateMutability === 'view') {
                    return await this.runRead();
                }

                if (this.isNative) {
                    return await this.runNative(opts);
                }

                return await this.runEVM(opts);
            } catch (e) {
                this.result = e.message;
            }

            this.endLoading();
        },
        getFunctionAbi() {
            return `${this.abi.name}(${this.abi.inputs.map(i => i.type).join(',')})`;
        },
        async getEthersFunction(provider) {
            const contractInstance = await this.contract.getContractInstance(provider);
            return contractInstance[this.getFunctionAbi()];
        },
        runRead() {
            let params;

            try {
                params = this.getFormattedParams();
                this.errorMessage = null;
            } catch (e) {
                this.errorMessage = e;
                return Promise.reject(e);
            }

            return this.getEthersFunction()
                .then(func => func(...params)
                    .then(response => {
                        this.result = response;
                        this.errorMessage = null;
                    })
                    .catch((msg) => {
                        this.errorMessage = msg;
                    })
                    .finally(() => this.endLoading()),
                );
        },
        async runNative(opts) {
            const contractInstance = await this.contract.getContractInstance();
            const func = contractInstance.populateTransaction[this.getFunctionAbi()];
            const gasEstimater = contractInstance.estimateGas[this.getFunctionAbi()];
            const gasLimit = await gasEstimater(...this.getFormattedParams(), Object.assign({from: this.address}, opts));
            const unsignedTrx = await func(...this.getFormattedParams(), opts);
            const nonce = parseInt(await this.$evm.telos.getNonce(this.address), 16);
            const gasPrice = BigNumber.from(`0x${await this.$evm.telos.getGasPrice()}`);
            unsignedTrx.nonce = nonce;
            unsignedTrx.gasLimit = gasLimit;
            unsignedTrx.gasPrice = gasPrice;

            // DO NOT INCLUDE CHAINID, EIP155 is only for replay attacks and you cannot replay a Telos native signed trx
            // this can however break stuff that trys to decode this trx
            //unsignedTrx.chainId = this.$evm.chainId;

            if (opts.value) {
                unsignedTrx.value = opts.value;
            }

            const raw = ethers.utils.serializeTransaction(unsignedTrx);

            let user = this.$providerManager.getProvider();
            await user.signTransaction(
                {
                    actions: [{
                        account: 'eosio.evm',
                        name: 'raw',
                        authorization: [
                            {
                                actor: this.nativeAccount,
                                permission: 'active',
                            },
                        ],
                        data: {
                            ram_payer: 'eosio.evm',
                            tx: raw.replace(/^0x/, ''),
                            estimate_gas: false,
                            sender: this.address.replace(/^0x/, '').toLowerCase(),
                        },
                    }],
                },
                {
                    blocksBehind: 3,
                    expireSeconds: 30,
                },
            );

            // This doesn't produce the right hash... but would be nice to use ethers here instead of ethereumjs/tx
            //  maybe just need to have signed transaction with an empty signature?  What is etherumjs/tx doing differently?
            //this.hash = ethers.utils.keccak256(raw);

            const trxBuffer = Buffer.from(raw.replace(/^0x/, ''), 'hex');

            const tx = Transaction.fromSerializedTx(trxBuffer, {
                common: this.$evm.chainConfig,
            });

            this.hash = `0x${tx.hash().toString('hex')}`;
            this.endLoading();
        },
        async runEVM(opts) {
            const func = await this.getEthersFunction(this.$providerManager.getEthersProvider().getSigner());

            let params;
            try {
                params = this.getFormattedParams();
                this.errorMessage = null;
            } catch (e) {
                this.errorMessage = e;

                throw e;
            }
            const result = await func(...params, opts);
            this.hash = result.hash;
            this.endLoading();
        },
        endLoading() {
            this.loading = false;
        },
    },
}
</script>

<style lang='sass'>
.amount-dialog.q-card
    padding: 1.5rem !important

.output-container
    margin: 0 1rem 1rem 1rem
    font-weight: 500
    font-size: .75rem

.run-button
    margin: 0 1rem 1rem 1rem
    border-radius: .25rem
</style>
