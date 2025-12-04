<template>
    <div class="top-app-bar sticky-top">
        <div class="top-app-bar-headline">
            <span class="title-large">{{ props.subreddit }}</span>
        </div>
        <div class="top-app-bar-trailing-icons">
            <div class="top-app-bar-trailing-icon-container position-relative" @touchstart.prevent="open_menu_options">
                <span class="top-app-bar-trailing-icon material-icons">more_vert</span>
                <div class="menu" v-show="menu_visible" @touchstart.prevent="close_menu_options">
                    <div class="menu-container">
                        <div class="menu-item" @touchstart.prevent="scroll_top">
                            <span class="material-icons">move_up</span>
                            <span class="label-large">Scroll to top</span>
                        </div>
                        <div class="menu-item" @touchstart.prevent="refresh">
                            <span class="material-icons">refresh</span>
                            <span class="label-large">Refresh</span>
                        </div>
                        <div class="menu-item" @touchstart.prevent="clear_hidden">
                            <span class="material-icons">restore</span>
                            <span class="label-large">Clear hidden posts</span>
                        </div>
                    </div>
                </div>
            </div>
        </div>
    </div>
    <div class="d-flex dps-16 dpb-16 md-background">
        <div class="chips-container" @touchstart.stop="open_dialog">
            <span class=" material-icons">sort</span>
            <span class="label-large">{{ display }}</span>
        </div>
    </div>
    <div class="dialog" v-show="dialog_visible" @touchstart.prevent="close_dialog">
        <div class="dialog-container" @touchstart.stop.prevent>
            <div class="dialog-title">
                <span class="headline-small">Sort options</span>
            </div>
            <div v-show="tab == 'sort'" class="list-container py-0">
                <div class="list-item" v-for="sort_type in sort_types" @touchstart.stop.prevent="change_sort(sort_type)">
                    <div class="list-item-leading-icon">
                        <span class="material-icons">{{ sort_type.icon }}</span>
                    </div>
                    <span class="body-large">{{ sort_type.name }}</span>
                    <div class="list-item-trailing-icon"
                        v-show="sort_type.value == 'top' || sort_type.value == 'controversial'">
                        <span class="material-icons">arrow_right</span>
                    </div>
                </div>
            </div>
            <div v-show="tab == 'time'" class="list-container py-0">
                <div class="list-item" v-for="time_type in time_types" @touchstart.stop.prevent="change_time(time_type)">
                    <div class="list-item-leading-icon">
                        <span class="material-icons">{{ time_type.icon }}</span>
                    </div>
                    <span class="body-large">{{ time_type.name }}</span>
                </div>
            </div>
        </div>
    </div>
</template>

<script setup>
import { ref, onBeforeMount } from 'vue';
import { clear_hidden } from '/js/store.js';
import { useRouter } from 'vue-router';

const router = useRouter();

const sort = ref("hot");
const time = ref("day");
const display = ref("Hot");

const tab = ref('sort');
const temp_sort = ref("hot");
const temp_display = ref("Hot");

const menu_visible = ref(false);
const dialog_visible = ref(false);

const emit = defineEmits(['params_changed'])

const props = defineProps({
	subreddit: {
		type: String,
		required: false
	}
})

defineExpose({
	sort,
	time
})

function loadSortPreferences() {
	const sortPrefs = localStorage.getItem('sort_preferences');
	if (!sortPrefs) return;

	try
	{
		const prefs = JSON.parse(sortPrefs);
		const homePrefs = prefs['home'];
		if (homePrefs)
		{
			sort.value = homePrefs.sort || "hot";
			time.value = homePrefs.time || "day";
			display.value = homePrefs.display || "Hot";
		}
	}
	catch (err)
	{
		console.error('Failed to load sort preferences:', err);
	}
}

function saveSortPreferences() {
	try
	{
		let sortPrefs = {};
		const existing = localStorage.getItem('sort_preferences');
		if (existing)
		{
			sortPrefs = JSON.parse(existing);
		}

		sortPrefs['home'] = {
			sort: sort.value,
			time: time.value,
			display: display.value
		};

		localStorage.setItem('sort_preferences', JSON.stringify(sortPrefs));
	}
	catch (err)
	{
		console.error('Failed to save sort preferences:', err);
	}
}

onBeforeMount(() => {
	loadSortPreferences();
})

const sort_types = [
    {
        name: "Best",
        value: "best",
        icon: "rocket_launch"
    },
    {
        name: "Hot",
        value: "hot",
        icon: "local_fire_department"
    },
    {
        name: "New",
        value: "new",
        icon: "schedule"
    },
    {
        name: "Rising",
        value: "rising",
        icon: "moving"
    },
    {
        name: "Controversial",
        value: "controversial",
        icon: "forum"
    },
    {
        name: "Top",
        value: "top",
        icon: "leaderboard"
    },
]

const time_types = [
    {
        name: "Past hour",
        value: "hour",
        icon: "history"
    },
    {
        name: "Past 24 hours",
        value: "day",
        icon: "today"
    },
    {
        name: "Past week",
        value: "week",
        icon: "date_range"
    },
    {
        name: "Past month",
        value: "month",
        icon: "calendar_month"
    },
    {
        name: "Past year",
        value: "year",
        icon: "event_note"
    },
    {
        name: "All time",
        value: "all",
        icon: "stars"
    }
]

async function open_menu_options(event) {
    event.stopPropagation();
    menu_visible.value = true;
}

async function close_menu_options(event) {
    event.stopPropagation();
    menu_visible.value = false;
}

async function open_dialog() {
    dialog_visible.value = true;
}

async function close_dialog() {
    dialog_visible.value = false;

    // reset to default
    tab.value = "sort";
    temp_sort.value = "hot";
    temp_display.value = "hot";
}

async function refresh() {
    router.go();
}

async function change_sort(sort_type) {
	display.value = sort_type.name;

	if (sort_type.value == 'top' || sort_type.value == 'controversial')
	{
		temp_display.value = sort_type.name;
		temp_sort.value = sort_type.value;
		tab.value = 'time';
		return
	}

	sort.value = sort_type.value;
	saveSortPreferences();
	emit('params_changed');
	close_dialog();
}

async function change_time(time_type) {
	display.value = temp_display.value + ' ' + time_type.name;
	sort.value = temp_sort.value;
	time.value = time_type.value;
	saveSortPreferences();
	emit('params_changed');
	close_dialog();
}

async function scroll_top() {
    let view = document.querySelector('.content-view');
    view.scroll({
        top: 0,
        behavior: 'smooth'
    });
}
</script>