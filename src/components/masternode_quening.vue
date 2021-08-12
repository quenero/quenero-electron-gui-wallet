<template>
<div class="masternode-quening">
    <div class="q-px-md q-pt-md">
        <QueneroField :label="$t('fieldLabels.masterNodeKey')" :error="$v.masternode.key.$error">
            <q-input v-model="masternode.key"
                :dark="theme=='dark'"
                @blur="$v.masternode.key.$touch"
                :placeholder="$t('placeholders.hexCharacters', { count: 64 })"
                hide-underline
            />
        </QueneroField>

        <QueneroField :label="$t('fieldLabels.amount')" class="q-mt-md" :error="$v.masternode.amount.$error">
            <q-input v-model="masternode.amount"
                :dark="theme=='dark'"
                type="number"
                min="0"
                :max="unlocked_balance / 1e12"
                placeholder="0"
                @blur="$v.masternode.amount.$touch"
                hide-underline
            />
            <q-btn color="secondary" @click="masternode.amount = unlocked_balance / 1e12" :text-color="theme=='dark'?'white':'dark'">
                {{ $t("buttons.all") }}
            </q-btn>
        </QueneroField>



        <q-field class="buttons q-pt-sm">
            <q-btn
                :disable="!is_able_to_send"
                color="primary" @click="supernode()" :label="$t('buttons.supernode')" />
            <q-btn
                :disable="!is_able_to_send"
                color="secondary" @click="sweepAllWarning()" :label="$t('buttons.sweepAll')" />
        </q-field>

    </div>

    <MasternodeUnlock />

    <q-inner-loading :visible="supernode_status.sending || tx_status.sending" :dark="theme=='dark'">
        <q-spinner color="primary" :size="30" />
    </q-inner-loading>
</div>
</template>

<script>
const { clipboard } = require("electron")
const objectAssignDeep = require("object-assign-deep");
import { mapState } from "vuex"
import { required, decimal } from "vuelidate/lib/validators"
import { i18n } from "plugins/i18n"
import { payment_id, masternode_key, greater_than_zero, address } from "src/validators/common"
import QueneroField from "components/quenero_field"
import WalletPassword from "src/mixins/wallet_password"
import MasternodeUnlock from "components/masternode_unlock"

export default {
    name: "MasternodeQuening",
    computed: mapState({
        theme: state => state.gateway.app.config.appearance.theme,
        unlocked_balance: state => state.gateway.wallet.info.unlocked_balance,
        info: state => state.gateway.wallet.info,
        supernode_status: state => state.gateway.masternode_status.supernode,
        tx_status: state => state.gateway.tx_status,
        award_address: state => state.gateway.wallet.info.address,
        is_ready (state) {
            return this.$store.getters["gateway/isReady"]
        },
        is_able_to_send (state) {
            return this.$store.getters["gateway/isAbleToSend"]
        },
        address_placeholder (state) {
            const wallet = state.gateway.wallet.info;
            const prefix = (wallet && wallet.address && wallet.address[0]) || "L";
            return `${prefix}..`;
        },
    }),
    data () {
        return {
            masternode: {
                key: "",
                amount: 0,
            },
        }
    },
    validations: {
        masternode: {
            key: { required, masternode_key },
            amount: {
                required,
                decimal,
                greater_than_zero,
            },
        }
    },
    watch: {
        supernode_status: {
            handler(val, old){
                if(val.code == old.code) return
                switch(this.supernode_status.code) {
                    case 0:
                        this.$q.notify({
                            type: "positive",
                            timeout: 1000,
                            message: this.supernode_status.message
                        })
                        this.$v.$reset();
                        this.masternode = {
                            key: "",
                            amount: 0,
                        }
                        break;
                    case -1:
                        this.$q.notify({
                            type: "negative",
                            timeout: 3000,
                            message: this.supernode_status.message
                        })
                        break;
                }
            },
            deep: true
        },
        tx_status: {
            handler(val, old){
                if(val.code == old.code) return
                switch(this.tx_status.code) {
                    case 0:
                        this.$q.notify({
                            type: "positive",
                            timeout: 1000,
                            message: this.tx_status.message
                        })
                        break;
                    case -1:
                        this.$q.notify({
                            type: "negative",
                            timeout: 3000,
                            message: this.tx_status.message
                        })
                        break;
                }
            },
            deep: true
        },
    },
    methods: {
        sweepAllWarning: function () {
            this.$q.dialog({
                title: this.$t("dialog.sweepAllWarning.title"),
                message: this.$t("dialog.sweepAllWarning.message"),
                ok: {
                    label: this.$t("dialog.sweepAllWarning.ok")
                },
                cancel: {
                    flat: true,
                    label: this.$t("dialog.buttons.cancel"),
                    color: this.theme === "dark" ? "white" : "dark"
                }
            }).then(() => {
                this.sweepAll()
            }).catch(() => {})
        },
        sweepAll: function () {
            const { unlocked_balance } = this.info;

            const tx = {
                amount: unlocked_balance / 1e12,
                address: this.award_address,
                priority: 0
            };

            this.showPasswordConfirmation({
                title: this.$t("dialog.sweepAll.title"),
                noPasswordMessage: this.$t("dialog.sweepAll.message"),
                ok: {
                    label: this.$t("dialog.sweepAll.ok")
                },
            }).then(password => {
                this.$store.commit("gateway/set_tx_status", {
                    code: 1,
                    message: "Sweeping all",
                    sending: true
                })
                const newTx = objectAssignDeep.noMutate(tx, {password})
                this.$gateway.send("wallet", "transfer", newTx)
            }).catch(() => {
            })
        },
        supernode: function () {
            this.$v.masternode.$touch()

            if (this.$v.masternode.key.$error) {
                this.$q.notify({
                    type: "negative",
                    timeout: 1000,
                    message: this.$t("notification.errors.invalidMasternodeKey")
                })
                return
            }

            if(this.masternode.amount < 0) {
                this.$q.notify({
                    type: "negative",
                    timeout: 1000,
                    message: this.$t("notification.errors.negativeAmount")
                })
                return
            } else if(this.masternode.amount == 0) {
                this.$q.notify({
                    type: "negative",
                    timeout: 1000,
                    message: this.$t("notification.errors.zeroAmount")
                })
                return
            } else if(this.masternode.amount > this.unlocked_balance / 1e12) {
                this.$q.notify({
                    type: "negative",
                    timeout: 1000,
                    message: this.$t("notification.errors.notEnoughBalance")
                })
                return
            } else if (this.$v.masternode.amount.$error) {
                this.$q.notify({
                    type: "negative",
                    timeout: 1000,
                    message: this.$t("notification.errors.invalidAmount")
                })
                return
            }

            this.showPasswordConfirmation({
                title: this.$t("dialog.supernode.title"),
                noPasswordMessage: this.$t("dialog.supernode.message"),
                ok: {
                    label: this.$t("dialog.supernode.ok")
                },
            }).then(password => {
                this.$store.commit("gateway/set_snode_status", {
                    supernode: {
                        code: 1,
                        message: "Quening...",
                        sending: true
                    }
                })
                const masternode = objectAssignDeep.noMutate(this.masternode, {
                    password,
                    destination: this.award_address
                })

                this.$gateway.send("wallet", "supernode", masternode)
            }).catch(() => {
            })
        }
    },
    mixins: [WalletPassword],
    components: {
        QueneroField,
        MasternodeUnlock
    }
}
</script>

<style lang="scss">
.masternode-quening {
    .buttons {
        .q-btn:not(:first-child) {
            margin-left: 8px;
        }
    }
}
</style>
