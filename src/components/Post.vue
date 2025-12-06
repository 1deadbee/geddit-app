<template>
    <div v-if="loading && !post" class="d-flex justify-content-center align-items-center" style="min-height: 200px; margin-top: 60px;">
        <div class="d-flex circle md-background p-2">
            <div class="spinner-border text-4" role="status"></div>
        </div>
    </div>
    <div v-if="error && !post" class="d-flex flex-column justify-content-center align-items-center" style="min-height: 200px; margin-top: 60px;">
        <span class="material-icons-outlined" style="font-size: 48px; color: var(--md-sys-color-error);">error_outline</span>
        <p class="mt-3 text-center px-4">Failed to load post. Please check your internet connection.</p>
        <button class="btn btn-primary mt-2" @click="retry">Retry</button>
    </div>
    <div v-if="post">
        <FullPost :post="post" />
        <div class="d-flex dpx-16">
            <span class="title-small text-4">Comments</span>
        </div>
        <div class="list-container dpy-16">
            <div v-for="(comment, index) in comments" :key="index">
                <div v-show="comment.kind == 't1' && !comment.hidden" class="list-item-full list-item-divider dpx-16" @click="toggleComment(index)">
                    <div v-show="comment.depth" class="comment-depth-container">
                        <div class="comment-depth" v-for="_ in comment.depth">
                            <div class="comment-depth-line"></div>
                        </div>
                    </div>
                    <div class="comment-body">
                        <div class="d-flex align-items-center dpb-4">
                            <div class="list-item-leading-icon ps-0 dpe-8">
                                <span class="material-icons">{{ comment.author == 'AutoModerator' ? 'local_police' : 'face'
                                }}</span>
                            </div>
                            <span class="label-small text-10" @click.passive="open_user(comment.author)">{{
                                comment.author }}</span>
                            <span v-if="comment.author == post.data.author" class="label-small op_indicator fw-bold dps-4">(OP)</span>
                            <span v-if="comment.collapsed" class="material-icons text-10 dps-4" style="font-size: 16px;">expand_more</span>
                        </div>
                        <span class="body-medium" :class="{ 'comment-collapsed': comment.collapsed }" v-html="markdown(comment.body)"></span>
                    </div>
                </div>
            </div>
        </div>
    </div>
</template>

<script setup>
import { ref, onBeforeMount, onActivated, nextTick } from 'vue';
import { useRouter } from 'vue-router';
import FullPost from './FullPost.vue';
import { Geddit } from "/js/geddit.js";

const router = useRouter();
const geddit = new Geddit();

const post = ref(null);
const comments = ref([]);
const view = ref(document.querySelector('.content-view'));

const loading = ref(false);
const error = ref(false);

async function setup() {
	loading.value = true;
	error.value = false;

	try
	{
		const timeoutPromise = new Promise((_, reject) =>
			setTimeout(() => reject(new Error('Request timeout')), 15000)
		);

		const fetchPromise = geddit.getSubmissionComments(router.currentRoute.value.params.id);

		let response = await Promise.race([fetchPromise, timeoutPromise]);

		if (!response)
		{
			error.value = true;
			return;
		}

		Promise.all(response.comments.map(async (comment) => {
			return await get_all_replies(comment);
		}))
			.then(replies => {
				comments.value = replies.flat();
			})

		post.value = response.submission;
		await nextTick();
	}
	catch (err)
	{
		console.error('Failed to load post:', err);
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

function markdown(body) {
    let html = decodeHtml(body);
    return convertLinksToImages(html);
}

async function open_user(author) {
    router.push(`/u/${author}`);
}

async function get_all_replies(comment, depth = 0) {
    let replies = [];
    if (comment.kind == "more") {
        replies.push({
            kind: "more",
            children: comment.data.children,
        })
        return replies;
    }

    replies.push({
        kind: "t1",
        author: comment.data.author,
        body: comment.data.body_html,
        depth: depth,
        collapsed: false,
    })

    if (!comment.data.replies) return replies;
    comment.data.replies.data.children.map(async (reply) => {
        replies.push(...await get_all_replies(reply, depth + 1));
    })
    return replies;
}

function toggleComment(index) {
	const comment = comments.value[index];
	comment.collapsed = !comment.collapsed;

	if (comment.collapsed) {
		const commentDepth = comment.depth;
		for (let i = index + 1; i < comments.value.length; i++)
		{
			if (comments.value[i].depth <= commentDepth)
			{
				break;
			}
			comments.value[i].hidden = true;
		}
	}
	else
	{
		const commentDepth = comment.depth;
		for (let i = index + 1; i < comments.value.length; i++)
		{
			if (comments.value[i].depth <= commentDepth)
			{
				break;
			}
			if (comments.value[i].depth === commentDepth + 1)
			{
				comments.value[i].hidden = false;
			}
		}
	}
}

onBeforeMount(() => {
    if (!router.currentRoute.value.params.id) {
        router.back();
        return;
    }

    setup();

    // Scroll to top
    view.value.scroll({
        top: 0
    })
})

onActivated(() => {
    // Scroll to the last position
    let pages = JSON.parse(localStorage.getItem("pages"));
    let this_page = pages.find(page => page.path == window.location.pathname);
    if (this_page) {
        view.value.scrollTop = parseInt(this_page.scroll);
    }
})
</script>