<template>
<div class="masternode-unlock" v-if="masternodes.length > 0">
    <div class="q-pa-md">
        <div class="q-pb-sm header">{{ $t('titles.currentlySNodes') }}</div>
        <q-list class="masternode-list" no-border>
            <q-item v-for="node in masternodes" :key="node.key">
                <q-item-main>
                    <q-item-tile class="ellipsis" label>{{ node.key }}</q-item-tile>
                    <q-item-tile sublabel class="non-selectable">{{ $t('strings.contribution') }}: <FormatQuenero :amount="node.amount" /></q-item-tile>
                </q-item-main>
                <q-item-side>
                    <q-btn
                        color="primary"
                        size="md"
                        :label="$t('buttons.unlock')"
                        :disabled="!is_ready || unlock_status.sending"
                        @click="unlockWarning(node.key)"
                    />
                </q-item-side>
                 <q-context-menu>
                    <q-list link separator style="min-width: 150px; max-height: 300px;">
                        <q-item v-close-overlay @click.native="copyKey(node.key, $event)">
                            <q-item-main :label="$t('menuItems.copyMasternodeKey')" />
                        </q-item>
                        <q-item v-close-overlay @click.native="openExplorer(node.key)">
                            <q-item-main :label="$t('menuItems.viewOnExplorer')" />
                        </q-item>
                    </q-list>
                </q-context-menu>
            </q-item>
        </q-list>
    </div>

    <q-inner-loading :visible="unlock_status.sending" :dark="theme=='dark'">
        <q-spinner color="primary" :size="30" />
    </q-inner-loading>
</div>
</template>

<script>
const objectAssignDeep = require("object-assign-deep");
import { mapState } from "vuex"
import { required } from "vuelidate/lib/validators"
import { masternode_key } from "src/validators/common"
import QueneroField from "components/quenero_field"
import WalletPassword from "src/mixins/wallet_password"
import FormatQuenero from "components/format_quenero"

export default {
    name: "MasternodeUnlock",
    computed: mapState({
        theme: state => state.gateway.app.config.appearance.theme,
        unlock_status: state => state.gateway.masternode_status.unlock,
        our_address: state => {
            const primary = state.gateway.wallet.address_list.primary[0]
            return (primary && primary.address) || null
        },
        is_ready (state) {
            return this.$store.getters["gateway/isReady"]
        },
        masternodes (state) {
            const nodes = state.gateway.daemon.masternodes
            const getContribution = node => node.contributors.find(c => c.address === this.our_address)
            // Only show nodes that we contributed to
            return nodes.filter(getContribution).map(n => {
                const ourContribution = getContribution(n)
                return {
                    key: n.masternode_pubkey,
                    amount: ourContribution.amount
                }
            })
        }
    }),
    validations: {
        node_key: { required, masternode_key }
    },
    watch: {
        unlock_status: {
            handler(val, old){
                if(val.code == old.code) return
                switch(this.unlock_status.code) {
                    case 0:
                        this.key = null
                        this.password = null

                        this.$q.notify({
                            type: "positive",
                            timeout: 1000,
                            message: this.unlock_status.message
                        })
                        this.$v.$reset();
                        break;
                    case 1:
                        // Tell the user to confirm
                         this.$q.dialog({
                            title: this.$t("dialog.unlockMasternode.confirmTitle"),
                            message: this.unlock_status.message,
                            ok: {
                                label: this.$t("dialog.unlockMasternode.ok")
                            },
                            cancel: {
                                flat: true,
                                label: this.$t("dialog.buttons.cancel"),
                                color: this.theme=="dark"?"white":"dark"
                            }
                        }).then(() => {
                            this.gatewayUnlock(this.password, this.key, true);
                        }).catch(() => {
                        })
                        break;
                    case -1:
                        this.key = null
                        this.password = null

                        this.$q.notify({
                            type: "negative",
                            timeout: 3000,
                            message: this.unlock_status.message
                        })
                        break;
                    default: break;
                }
            },
            deep: true
        },
    },
    methods: {
        unlockWarning (key) {
            this.$q.dialog({
                title: this.$t("dialog.unlockMasternodeWarning.title"),
                message: this.$t("dialog.unlockMasternodeWarning.message"),
                ok: {
                    label: this.$t("dialog.unlockMasternodeWarning.ok")
                },
                cancel: {
                    flat: true,
                    label: this.$t("dialog.buttons.cancel"),
                    color: this.theme === "dark" ? "white" : "dark"
                }
            }).then(() => {
                this.unlock(key)
            }).catch(() => {})
        },
        unlock (key) {
            // We store this as it could change between the 2 step process
            this.key = key

            this.showPasswordConfirmation({
                title: this.$t("dialog.unlockMasternode.title"),
                noPasswordMessage: this.$t("dialog.unlockMasternode.message"),
                ok: {
                    label: this.$t("dialog.unlockMasternode.ok")
                },
            }).then(password => {
                this.password = password
                this.gatewayUnlock(this.password, this.key, false);
            }).catch(() => {
            })
        },
        gatewayUnlock (password, key, confirmed = false) {
            this.$store.commit("gateway/set_snode_status", {
                unlock: {
                    code: 2, // Code 1 is reserved for can_unlock
                    message: "Unlocking...",
                    sending: true
                }
            })
            this.$gateway.send("wallet", "unlock_supernode", {
                password,
                masternode_key: key,
                confirmed
            })
        },
        copyKey (key, event) {
            event.stopPropagation()
            for(let i = 0; i < event.path.length; i++) {
                if(event.path[i].tagName == "BUTTON") {
                    event.path[i].blur()
                    break
                }
            }
            clipboard.writeText(key)
            this.$q.notify({
                type: "positive",
                timeout: 1000,
                message: this.$t("notification.positive.copied", { item: "Masternode key" })
            })
        },
        openExplorer (key) {
            this.$gateway.send("core", "open_explorer", {type: "masternode", id: key})
        }
    },

    mixins: [WalletPassword],
    components: {
        QueneroField,
        FormatQuenero
    }
}
</script>

<style lang="scss">
.masternode-unlock {
    user-select: none;
    .header {
        font-weight: 450;
    }
    .q-item-sublabel, .q-list-header {
        font-size: 14px;
    }
}
</style>
