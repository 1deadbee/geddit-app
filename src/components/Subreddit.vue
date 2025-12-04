<template>
    <div v-if="loading && !data" class="d-flex justify-content-center align-items-center" style="min-height: 200px; margin-top: 60px;">
        <div class="d-flex circle md-background p-2">
            <div class="spinner-border text-4" role="status"></div>
        </div>
    </div>
    <div v-if="error && !data" class="d-flex flex-column justify-content-center align-items-center" style="min-height: 200px; margin-top: 60px;">
        <span class="material-icons-outlined" style="font-size: 48px; color: var(--md-sys-color-error);">error_outline</span>
        <p class="mt-3 text-center px-4">Failed to load subreddit. Please check your internet connection.</p>
        <button class="btn btn-primary mt-2" @click="retry">Retry</button>
    </div>
    <div v-if="data">
        <TopAppBarSubreddit ref="topbar" :subreddit="data.display_name_prefixed" @params_changed="params_changed" />
        <div class="d-flex flex-column p-3 pt-0">
            <div class="banner d-flex justify-content-center align-items-center position-relative mb-2">
                <img v-show="data.banner_img" :src="data.banner_img" class="cover vh-25 rounded">
                <img v-show="icon" :src="icon" class="snoovatar position-absolute">
            </div>
            <div class="d-flex flex-column align-items-center mb-2">
                <span class="title-large text-center text-4">{{ data.title }}</span>
                <span class="title-medium text-center text-4">{{ data.display_name_prefixed }}</span>
                <hr class="text-4 w-100 m-0 my-2">
                <span class="label-medium text-center text-4">{{ get_description() }}</span>
            </div>
            <div class="d-flex align-items-center justify-content-center mb-3">
                <span class="label-medium text-10">
                    {{ format_subscribers() }}
                    <span class="text-4">members</span>
                </span>
                <span class="label-medium dmx-4 text-4">-</span>
                <span class="label-medium text-10">
                    {{ format_active() }}
                    <span class="text-4">online</span>
                </span>
            </div>
            <div class="d-flex justify-content-center align-items-center">
                <div class="md-filled-button-with-icon bg-10 text-4" v-show="!followed" @click.passive="follow">
                    <span class="material-icons">bookmark_add</span>
                    <span class="label-large">Follow</span>
                </div>
                <div class="md-filled-button-with-icon bg-3 text-4" v-show="followed" @click.passive="unfollow">
                    <span class="material-icons">bookmark_added</span>
                    <span class="label-large">Following</span>
                </div>
            </div>
        </div>
        <div class="cards dpb-16">
            <Post v-for="post in posts" :post="post.data" />
        </div>
        <div v-show="!scroll_loaded" class="md-progress-container">
            <div class="md-progress">
                <div class="md-progress-indicator"></div>
            </div>
        </div>
    </div>
</template>

<script setup>
import { ref, onBeforeMount, onActivated, onDeactivated } from 'vue';
import { useRouter } from 'vue-router';
import { Geddit } from "/js/geddit.js";
import Post from './CompactPost.vue';
import TopAppBarSubreddit from './TopAppBarSubreddit.vue';

const router = useRouter();
const geddit = new Geddit();
const topbar = ref(null);

const posts = ref([]);
const after = ref(null);

const data = ref(null);
const icon = ref(null);
const followed = ref(false);

const scroll_loaded = ref(true);
const loading = ref(false);
const error = ref(false);

function get_subreddit_key() {
	const id = router.currentRoute.value.params.id;
	if (!id) return 'unknown';
	return id.toLowerCase();
}

function get_subreddit_sort_preferences() {
	try
	{
		const prefs = JSON.parse(localStorage.getItem('sort_preferences'));
		if (prefs)
		{
			const key = get_subreddit_key();
			if (prefs[key])
			{
				return {
					sort: prefs[key].sort || "hot",
					time: prefs[key].time || "day"
				};
			}
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

async function params_changed() {
    posts.value = [];
    after.value = null;
    scroll_loaded.value = true;
    get_posts();
}

async function check_followed() {
    if (!data.value) return;
    let subs = JSON.parse(localStorage.getItem("subreddits"));
    if (subs.find(sub => sub.display_name == data.value.display_name)) {
        followed.value = true;
    }
}

async function follow() {
    let subs = JSON.parse(localStorage.getItem("subreddits"));
    subs.push({
        display_name: data.value.display_name,
        public_description: data.value.public_description.slice(0, 42) + '…',
        subscribers: data.value.subscribers,
        community_icon: icon.value,
    });
    localStorage.setItem("subreddits", JSON.stringify(subs));
    followed.value = true;
}

async function unfollow() {
    let subs = JSON.parse(localStorage.getItem("subreddits"));
    subs = subs.filter(sub => sub.display_name != data.value.display_name);
    localStorage.setItem("subreddits", JSON.stringify(subs));
    followed.value = false;
}

async function get_subreddit() {
	loading.value = true;
	error.value = false;

	try
	{
		const timeoutPromise = new Promise((_, reject) =>
			setTimeout(() => reject(new Error('Request timeout')), 15000)
		);

		const fetchPromise = geddit.getSubreddit(router.currentRoute.value.params.id);

		let response = await Promise.race([fetchPromise, timeoutPromise]);

		if (!response)
		{
			error.value = true;
			return;
		}

		data.value = response;
		check_followed();
		get_sub_icon();
	}
	catch (err)
	{
		console.error('Failed to load subreddit:', err);
		error.value = true;
	}
	finally
	{
		loading.value = false;
	}
}

async function get_sub_icon() {
    if (data.value.icon_img) {
        icon.value = data.value.icon_img;
        return
    }

    if (data.value.community_icon) {
        icon.value = data.value.community_icon.split("?")[0];
        return
    }

    icon.value = "/images/subreddit.svg";
}

function format_subscribers() {
    if (!data.value.subscribers) {
        return "private";
    }

    return data.value.subscribers.toLocaleString();
}

function format_active() {
    if (data.value.active_user_count) {
        return data.value.active_user_count.toLocaleString();
    }
    return "private";
}

function get_description() {
    let txt = document.createElement('textarea');
    txt.innerHTML = data.value.public_description;
    return txt.value;
}

async function setup() {
	try
	{
		const timeoutPromise = new Promise((_, reject) =>
			setTimeout(() => reject(new Error('Request timeout')), 15000)
		);

		const { sort: savedSort, time: savedTime } = get_subreddit_sort_preferences();

		const fetchPromise = geddit.getSubmissions(savedSort, router.currentRoute.value.params.id, {
			t: savedTime
		});

		let response = await Promise.race([fetchPromise, timeoutPromise]);

		if (!response) return;

		posts.value = response.posts;
		after.value = response.after;
	}
	catch (err)
	{
		console.error('Failed to load posts:', err);
	}
}

async function get_posts() {
	try
	{
		const timeoutPromise = new Promise((_, reject) =>
			setTimeout(() => reject(new Error('Request timeout')), 15000)
		);

		const fetchPromise = geddit.getSubmissions(topbar.value.sort, router.currentRoute.value.params.id, {
			t: topbar.value.time
		});

		let response = await Promise.race([fetchPromise, timeoutPromise]);

		if (!response) return;

		posts.value = response.posts;
		after.value = response.after;
	}
	catch (err)
	{
		console.error('Failed to load posts:', err);
	}
}

function retry() {
	get_subreddit();
	setup();
}

async function scroll() {
    scroll_loaded.value = false;

    let response = null;

    response = await geddit.getSubmissions(topbar.value.sort, router.currentRoute.value.params.id, {
        after: after.value,
        t: topbar.value.time
    });

    if (!response || !response.posts.length) {
        after.value = null;
        scroll_loaded.value = true;
        return;
    }

    posts.value.push(...response.posts);
    after.value = response.after;

    scroll_loaded.value = true;
}


function scroll_handle(el) {
    if (el.target.scrollTop + el.target.clientHeight >= el.target.scrollHeight - window.innerWidth && scroll_loaded.value && after.value) {
        scroll();
    }
}

async function handle_sort_route() {
    let changed = false;
    let time = router.currentRoute.value.query.t;
    let sort = router.currentRoute.value.params.sort;

    if (sort && sort != topbar.value.sort) {
        topbar.value.sort = sort;
        changed = true;
    }

    if (time && time != topbar.value.time) {
        topbar.value.time = time;
        changed = true;
    }

    if (!changed) return

    get_posts();
}

onBeforeMount(() => {
    if (!router.currentRoute.value.params.id) {
        router.back();
        return;
    }

    get_subreddit();
    setup();
})

onActivated(() => {
    check_followed();

    // Add the scroll event listener
    let view = document.querySelector('.content-view');
    view.addEventListener('scroll', scroll_handle)

    // Scroll to the last position
    let pages = JSON.parse(localStorage.getItem("pages"));
    let this_page = pages.find(page => page.path == window.location.pathname);
    if (this_page) {
        document.querySelector('.content-view').scrollTop = parseInt(this_page.scroll);
    }

    handle_sort_route();
})

onDeactivated(() => {
    // Disable scroll event listener
    let view = document.querySelector('.content-view');
    view.removeEventListener('scroll', scroll_handle);
})
</script>