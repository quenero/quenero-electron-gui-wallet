<template>
<q-page class="masternode-page">
    <div class="header row items-center justify-center q-pt-md">
        <q-btn-toggle
            v-model="screen"
            toggle-color="primary"
            color="secondary"
            :options="[
                {label: $t('titles.masterNode.quening'), value: 'quening'},
                {label: $t('titles.masterNode.registration'), value: 'registration'}
            ]"
        />
    </div>
    <MasternodeQuening v-if="screen === 'quening'"/>
    <MasternodeRegistration v-if="screen === 'registration'" />
</q-page>
</template>

<script>
import MasternodeQuening from "components/masternode_quening"
import MasternodeRegistration from "components/masternode_registration"
export default {
    data () {
        return {
            screen: "quening",
        }
    },
    components: {
        MasternodeQuening,
        MasternodeRegistration
    }
}
</script>

<style lang="scss">
</style>
