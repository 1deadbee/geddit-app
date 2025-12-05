<template>
	<div class="d-flex cover-50 position-relative md-background md-rounded-12" :style="image_options.style">
		<img class="position-relative cover-50 ct" loading="lazy" v-if="!fallback" :src="image_options.preview"
			@click.passive="open_link" @error="fallback = true">
		<div class="w-100 d-flex flex-fill ratio-16-9 md-background md-rounded-12 p-3 justify-content-center align-items-center ct"
			v-if="fallback" @click.passive="open_link">
			<p class="w-100 word-break fw-bold text-4 p-3 m-0">{{ props.data.url }}</p>
		</div>
		<div class="position-absolute bottom-0 end-0 m-2 link-indicator">
			<span class="material-icons text-4">open_in_new</span>
		</div>
	</div>
</template>

<script setup>
import { ref } from 'vue';
import { Browser } from '@capacitor/browser';

const props = defineProps({
    data: {
        type: Object,
        required: true
    }
})

const image_options = ref({});
const fallback = ref(false);

async function get_sources() {
	if (!props.data.preview) {
		if (props.data.thumbnail && props.data.thumbnail !== 'self' && props.data.thumbnail !== 'default' && props.data.thumbnail.startsWith('http')) {
			image_options.value = {
				preview: props.data.thumbnail,
				style: {
					'aspect-ratio': '16 / 9'
				}
			}
			return;
		}
		fallback.value = true;
		image_options.value = {
			style: {
				'aspect-ratio': '16 / 9'
			}
		}
		return;
	}

	if (!props.data.preview.images || !props.data.preview.images[0] || !props.data.preview.images[0].resolutions || props.data.preview.images[0].resolutions.length === 0) {
		fallback.value = true;
		image_options.value = {
			style: {
				'aspect-ratio': '16 / 9'
			}
		}
		return;
	}

	image_options.value = {
		preview: props.data.preview.images[0].resolutions.pop().url.replaceAll("&amp;", "&"),
		style: {
			'aspect-ratio': `${props.data.preview.images[0].resolutions.slice(-1)[0].width} / ${props.data.preview.images[0].resolutions.slice(-1)[0].height}`
		}
	}
}

async function open_link() {
    let pref = JSON.parse(localStorage.getItem("in_app_browser"));
    if (pref) {
        await Browser.open({
            url: props.data.url
        })
        return
    }
    window.open(props.data.url, '_blank')
}

// setup
get_sources();
</script>