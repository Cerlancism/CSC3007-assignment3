<template>
    <div>
        <v-container style="max-width: 1360px;">
            <v-row>
                <v-col class="sm-12">
                    <Choropleth />
                </v-col>
            </v-row>
        </v-container>
    </div>
</template>

<script lang="ts">
    import { Component, Vue } from 'vue-property-decorator';
    import { sleepAsync } from '@/utils/common'

    import Choropleth from '@/components/Choropleth.vue'

    @Component({
        components: {
            Choropleth
        },
    })
    export default class Home extends Vue
    {
        async mounted()
        {
            return
        }

        unmounted()
        {
            console.log(`unmounted ${this.constructor.name}`)
        }
    }
</script>
