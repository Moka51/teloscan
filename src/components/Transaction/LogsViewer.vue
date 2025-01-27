<template>
<div>
    <div v-if="logs.length === 0" class="row">
        <div class="col-12 u-flex--center">
            <q-icon class="fa fa-info-circle q-mr-md" size="md" />
            <h3>No logs found</h3>
        </div>
    </div>
    <div v-else class="row">
        <div class="col-12 u-flex--center-y">
            <q-toggle
                v-model="human_readable"
                icon="visibility"
                color="secondary"
                size="lg"
            />
            Human-readable logs
            <small v-if="!allVerified">
                <q-icon name="info" class="q-mb-xs q-ml-xs" size="14px"/>
                <q-tooltip>Verify the related contract for each log to see its human readable version</q-tooltip>
            </small>
        </div>
        <div class="col-12">
            <logs-table
                v-if="human_readable"
                :rawLogs="logs"
                :logs="parsedLogs"
                :allVerified="allVerified"
                :contract="contract"
            />
            <json-viewer
                v-else
                :value="logs"
                theme="custom-theme"
                class="q-mb-md"
            />
        </div>
    </div>
</div>
</template>

<script>
import JsonViewer from 'vue-json-viewer'
import LogsTable from 'components/Transaction/LogsTable'
import { TRANSFER_SIGNATURES } from 'src/lib/abi/signature/transfer_signatures';
import { BigNumber } from 'ethers';

export default {
    name: 'LogsViewer',
    components: {
        JsonViewer,
        LogsTable,
    },
    methods: {
        async getLogContract(log, type){
            try {
                return  await this.$contractManager.getContract(log.address.toLowerCase(), type);
            } catch (e) {
                console.error(`Failed to retrieve contract with address ${log.address}`);
            }
        },
    },
    props: {
        contract : {
            type: Object,
            required: false,
        },
        logs: {
            type: Array,
            required: true,
        },
    },

    created() {
        let verified = 0;

        this.logs.forEach(async (log) => {
            let contract;
            const function_signature = log.topics[0].substr(0, 10);
            if(TRANSFER_SIGNATURES.includes(function_signature)) {
                contract = await this.getLogContract(log, (log.topics.length === 4) ? 'erc721': 'erc20');
            } else {
                contract = await this.getLogContract(log);
            }
            if (contract){
                verified = (contract.isVerified()) ? verified + 1: verified;
                let parsedLog = await contract.parseLogs([log]);
                this.parsedLogs.push(parsedLog[0]);
                this.parsedLogs.sort((a,b) => BigNumber.from(a.logIndex).sub(BigNumber.from(b.logIndex)).toNumber());
            }
        });

        this.allVerified = (verified === this.logs.length);
    },
    data: () => ({
        human_readable: true,
        parsedLogs: [],
        allVerified: false,
    }),
}
</script>
