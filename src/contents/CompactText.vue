<template>
    <div v-show="text" class="d-flex cover-25 text-wrap text-break text-truncate overflow-hidden md-rounded-12 ct"
        @click.prevent="emit('open_post')">
        <div class="text-4 text-post" v-html="text" />
    </div>
</template>

<script setup>
import { ref } from 'vue';

const text = ref(null);
const emit = defineEmits(['open_post']);
const props = defineProps({
    data: {
        type: Object,
        required: true
    }
})

function decodeHtml(html) {
    let txt = document.createElement("textarea");
    txt.innerHTML = html;
    return txt.value;
}

function convertLinksToImages(html) {
	const parser = new DOMParser();
	const doc = parser.parseFromString(html, 'text/html');
	const links = doc.querySelectorAll('a[href*="preview.redd.it"]');

	links.forEach(link => {
		const href = link.getAttribute('href');
		if (href && (href.match(/\.(jpg|jpeg|png|gif|webp)/) || href.includes('preview.redd.it')))
		{
			const img = document.createElement('img');
			img.src = href;
			img.style.width = '100%';
			img.style.height = 'auto';
			link.replaceWith(img);
		}
	});

	return doc.body.innerHTML;
}

async function get_sources() {
    let html = decodeHtml(props.data.selftext_html);
    text.value = convertLinksToImages(html);
}

// setup
get_sources();
</script>