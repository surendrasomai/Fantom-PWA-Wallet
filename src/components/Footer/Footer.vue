<template>
    <footer class="f-footer">
        <div class="narrow-container">
            <div class="row align-items-center">
                <div class="col align-center-sm">
                    <social-media-links></social-media-links>
                </div>
                <div class="col align-center-sm align-end">
                    <a href="#" target="_blank" rel="nofollow">©2022 ANB Foundation </a>
                </div>
            </div>
        </div>
    </footer>
</template>

<script>
import SocialMediaLinks from '../SocialMediaLinks/SocialMediaLinks.vue';

export default {
    components: {
        SocialMediaLinks,
    },
};
</script>

<style lang="scss">
@import 'style';
</style>
