<template>
    <div class="wrapper">
        <div v-for="(image, index) in images" :key="index">
            <img :src="image.src" @click.prevent="toggle_controls">
            <div class="position-absolute top-0 end-0 m-2">
                <div class="d-flex position-relative">
                    <div class="position-absolute background cover-all opacity-75 rounded"></div>
                    <small class="gallery-indicator position-relative text-4 px-1" :length="images.length"></small>
                </div>
            </div>
            <div class="position-absolute bottom-0 end-0 z-1 m-3" v-show="controls_visible">
                <div class="md-fab md-foreground el-3" @click.passive="download(image.src, index)">
                    <span class="material-icons text-4">download</span>
                </div>
            </div>
        </div>
    </div>
</template>

<script setup>
import { onMounted, ref } from 'vue';
import { Filesystem, Directory } from '@capacitor/filesystem';
import { Toast } from '@capacitor/toast';

const emit = defineEmits(['open_post']);
const props = defineProps({
    data: {
        type: Object,
        required: true
    }
})

const images = ref([]);
const controls_visible = ref(false);

async function toggle_controls() {
    controls_visible.value = !controls_visible.value;
}

async function download(imageUrl, index) {
    let filename = `${Date.now()}_${props.data.id}_${index}`;

    let extension = new URL(imageUrl).pathname.split('.').pop();

    Filesystem.downloadFile({
        url: imageUrl,
        directory: Directory.Documents,
        path: `/geddit/${filename}.${extension}`
    }).then((res) => {
        console.log(res);
        Toast.show({
            text: 'Image saved to gallery',
            duration: 'short'
        });
    }).catch((error) => {
        console.error(error);
    })
}

async function get_sources() {
    if (!props.data.gallery_data) return;

    let order = props.data.gallery_data.items.map(item => item.media_id);
    let items = order.map(id => props.data.media_metadata[id]);

    images.value = items.map(item => ({
        src: item.s.u ? item.s.u.split("?")[0].replace("preview", "i") : item.s.gif
    }))
}

// setup
get_sources();

onMounted(() => {
    let wrapper = document.querySelector(".wrapper");
    wrapper.scrollLeft = 0;
})
</script>