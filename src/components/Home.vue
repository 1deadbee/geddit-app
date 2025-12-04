<template>
    <TopAppBar ref="topbar" subreddit="Popular" @params_changed="params_changed" />
    <div v-if="loading && !posts.length" class="d-flex justify-content-center align-items-center" style="min-height: 200px; margin-top: 60px;">
        <div class="d-flex circle md-background p-2">
            <div class="spinner-border text-4" role="status"></div>
        </div>
    </div>
    <div v-if="error && !posts.length" class="d-flex flex-column justify-content-center align-items-center" style="min-height: 200px; margin-top: 60px;">
        <span class="material-icons-outlined" style="font-size: 48px; color: var(--md-sys-color-error);">error_outline</span>
        <p class="mt-3 text-center px-4">Failed to load posts. Please check your internet connection.</p>
        <button class="btn btn-primary mt-2" @click="retry">Retry</button>
    </div>
    <div class="cards dpb-16">
        <Post v-for="post in posts" :post="post.data" />
    </div>
    <div v-show="!scroll_loaded" class="md-progress-container">
        <div class="md-progress">
            <div class="md-progress-indicator"></div>
        </div>
    </div>
</template>

<script setup>
import { ref, onBeforeMount, onActivated, onDeactivated } from 'vue';
import { Geddit } from "/js/geddit.js";
import Post from './CompactPost.vue';
import TopAppBar from './TopAppBar.vue';

const geddit = new Geddit();
const topbar = ref(null);

const posts = ref([]);
const after = ref(null);

const scroll_loaded = ref(true);
const loading = ref(false);
const error = ref(false);

function get_home_sort_preferences() {
	try
	{
		const prefs = JSON.parse(localStorage.getItem('sort_preferences'));
		if (prefs && prefs.home)
		{
			return {
				sort: prefs.home.sort || "hot",
				time: prefs.home.time || "day"
			};
		}
	}
	catch (err)
	{
		console.error('Failed to parse sort preferences:', err);
	}

	return {
		sort: "hot",
		time: "day"
	};
}

async function setup() {
	loading.value = true;
	error.value = false;

	try
	{
		const timeoutPromise = new Promise((_, reject) =>
			setTimeout(() => reject(new Error('Request timeout')), 15000)
		);

		const { sort: savedSort, time: savedTime } = get_home_sort_preferences();

		const fetchPromise = geddit.getSubmissions(savedSort, "popular", {
			t: savedTime
		});

		let response = await Promise.race([fetchPromise, timeoutPromise]);

		if (!response)
		{
			error.value = true;
			return;
		}

		posts.value = response.posts;
		after.value = response.after;
	}
	catch (err)
	{
		console.error('Failed to load posts:', err);
		error.value = true;
	}
	finally
	{
		loading.value = false;
	}
}

async function get_posts() {
	loading.value = true;
	error.value = false;

	try
	{
		const timeoutPromise = new Promise((_, reject) =>
			setTimeout(() => reject(new Error('Request timeout')), 15000)
		);

		const fetchPromise = geddit.getSubmissions(topbar.value.sort, "popular", {
			t: topbar.value.time
		});

		let response = await Promise.race([fetchPromise, timeoutPromise]);

		if (!response)
		{
			error.value = true;
			return;
		}

		posts.value = response.posts;
		after.value = response.after;
	}
	catch (err)
	{
		console.error('Failed to load posts:', err);
		error.value = true;
	}
	finally
	{
		loading.value = false;
	}
}

function retry() {
	setup();
}

async function scroll() {
    scroll_loaded.value = false;

    let response = await geddit.getSubmissions(topbar.value.sort, "popular", {
        after: after.value,
        t: topbar.value.time
    });
    if (!response || !response.posts.length) {
        after.value = null;
        scroll_loaded.value = true;
        return
    }

    posts.value.push(...response.posts);
    after.value = response.after;

    scroll_loaded.value = true;
}

async function params_changed() {
    posts.value = [];
    after.value = null;
    scroll_loaded.value = true;
    get_posts();
}

function scroll_handle(el) {
    if (el.target.scrollTop + el.target.clientHeight >= el.target.scrollHeight - window.innerWidth && scroll_loaded.value && after.value) {
        scroll();
    }
}

onBeforeMount(() => {
    setup();
})

onActivated(() => {
    // Add the scroll event listener
    let view = document.querySelector('.content-view');
    view.addEventListener('scroll', scroll_handle)

    // Scroll to the last position
    let pages = JSON.parse(localStorage.getItem("pages"));
    let this_page = pages.find(page => page.path == window.location.pathname);
    if (this_page) {
        document.querySelector('.content-view').scrollTop = parseInt(this_page.scroll);
    }
})

onDeactivated(() => {
    // Disable scroll event listener
    let view = document.querySelector('.content-view');
    view.removeEventListener('scroll', scroll_handle);
})
</script>