<template>
    <div class="about">
        <v-container style="max-width: 1280px">
            <MainBody
                msg="A choropleth visualization shows off which subsdistricts have the most population in SG."
            />
        </v-container>
    </div>
</template>

<script lang="ts">
    import { Component, Vue } from 'vue-property-decorator';
    import MainBody from '@/components/MainBody.vue'; // @ is an alias to /src

    @Component({
        components: {
            MainBody,
        },
    })
    export default class AboutView extends Vue
    {

    }
</script>