<!--
SPDX-FileCopyrightText: syuilo and misskey-project
SPDX-License-Identifier: AGPL-3.0-only
-->

<template>
<MkContainer :max-height="300" :foldable="true">
	<template #icon><i class="ti ti-photo"></i></template>
	<template #header>{{ i18n.ts.files }}</template>
	<div :class="$style.root">
		<MkLoading v-if="fetching"/>
		<div v-if="!fetching && files.length > 0" :class="$style.stream">
			<template v-for="file in files" :key="file.note.id + file.file.id">
				<div v-if="!showingFiles.includes(file.file.id)" :class="$style.img" @click="e=>onClick(e,file.file)" @dblclick="e=>onDblClick(file.file)">
					<!-- TODO: 画像以外のファイルに対応 -->
					<ImgWithBlurhash :class="$style.sensitiveImg" :hash="file.file.blurhash" :src="thumbnail(file.file)" :title="file.file.name" :forceBlurhash="true"/>
					<div :class="$style.hiddenTextWrapper">
						<div>
							<div v-if="file.file.isSensitive"><i class="ti ti-eye-exclamation"></i> {{ i18n.ts.sensitive }}{{ defaultStore.state.dataSaver.media ? ` (${i18n.ts.image}${file.file.size ? ' ' + bytes(file.file.size) : ''})` : '' }}</div>
							<div v-else><i class="ti ti-photo"></i> {{ defaultStore.state.dataSaver.media && file.file.size ? bytes(file.file.size) : i18n.ts.image }}</div>
							<div>{{ clickToShowMessage }}</div>
						</div>
					</div>
				</div>
				<MkA v-else :class="$style.img" :to="notePage(file.note)">
					<!-- TODO: 画像以外のファイルに対応 -->
					<ImgWithBlurhash
						:hash="file.file.blurhash"
						:src="thumbnail(file.file)"
						:title="file.file.name"
						@mouseover="defaultStore.state.showingAnimatedImages === 'interaction' ? playAnimation = true : ''"
						@mouseout="defaultStore.state.showingAnimatedImages === 'interaction' ? playAnimation = false : ''"
						@touchstart="defaultStore.state.showingAnimatedImages === 'interaction' ? playAnimation = true : ''"
						@touchend="defaultStore.state.showingAnimatedImages === 'interaction' ? playAnimation = false : ''"
					/>
				</MkA>
			</template>
		</div>
		<p v-if="!fetching && files.length == 0" :class="$style.empty">{{ i18n.ts.nothing }}</p>
	</div>
</MkContainer>
</template>

<script lang="ts" setup>
import { computed, onMounted, onUnmounted, ref, watch } from 'vue';
import * as Misskey from 'cherrypick-js';
import bytes from '@/filters/bytes.js';
import { getStaticImageUrl } from '@/scripts/media-proxy.js';
import { notePage } from '@/filters/note.js';
import { misskeyApi } from '@/scripts/misskey-api.js';
import MkContainer from '@/components/MkContainer.vue';
import ImgWithBlurhash from '@/components/MkImgWithBlurhash.vue';
import MkA from '@/components/global/MkA.vue';
import MkLoading from '@/components/global/MkLoading.vue';
import { defaultStore } from '@/store.js';
import { i18n } from '@/i18n.js';
import { confirmR18, wasConfirmR18 } from '@/scripts/check-r18';
import MkRippleEffect from '@/components/MkRippleEffect.vue';
import * as os from '@/os.js';

const props = defineProps<{
	user: Misskey.entities.UserDetailed;
}>();

const fetching = ref(true);
const files = ref<{
	note: Misskey.entities.Note;
	file: Misskey.entities.DriveFile;
}[]>([]);
const showingFiles = ref<string[]>([]);
watch(() => files, () => {
	for (const file of files.value) {
		if (defaultStore.state.nsfw === 'force' || defaultStore.state.dataSaver.media) {
			//nop
		} else if (file.file.isSensitive) {
			if (defaultStore.state.nsfw !== 'ignore') {
				//nop
			} else {
				if (wasConfirmR18()) {
					showingFiles.value.push(file.file.id);
				}
			}
		} else {
			showingFiles.value.push(file.file.id);
		}
	}
}, {
	deep: true,
	immediate: true,
});

const clickToShowMessage = computed(() => defaultStore.state.nsfwOpenBehavior === 'click'
	? i18n.ts.clickToShow
	// eslint-disable-next-line @typescript-eslint/no-unnecessary-condition
	: defaultStore.state.nsfwOpenBehavior === 'doubleClick'
		? i18n.ts.doubleClickToShow
		: '',
);
const playAnimation = ref(true);
if (defaultStore.state.showingAnimatedImages === 'interaction') playAnimation.value = false;
let playAnimationTimer = setTimeout(() => playAnimation.value = false, 5000);

function thumbnail(image: Misskey.entities.DriveFile): string | null {
	return (defaultStore.state.disableShowingAnimatedImages || defaultStore.state.dataSaver.media) || (['interaction', 'inactive'].includes(<string>defaultStore.state.showingAnimatedImages) && !playAnimation.value)
		? getStaticImageUrl(image.url)
		: image.thumbnailUrl;
}

function resetTimer() {
	playAnimation.value = true;
	clearTimeout(playAnimationTimer);
	playAnimationTimer = setTimeout(() => playAnimation.value = false, 5000);
}

onMounted(() => {
	if (defaultStore.state.showingAnimatedImages === 'inactive') {
		window.addEventListener('mousemove', resetTimer);
		window.addEventListener('touchstart', resetTimer);
		window.addEventListener('touchend', resetTimer);
	}

	misskeyApi('users/notes', {
		userId: props.user.id,
		withFiles: true,
		limit: 15,
	}).then(notes => {
		for (const note of notes) {
			if (!note.files) continue;
			for (const file of note.files) {
				files.value.push({
					note,
					file,
				});
			}
		}
		fetching.value = false;
	});
});

onUnmounted(() => {
	if (defaultStore.state.showingAnimatedImages === 'inactive') {
		window.removeEventListener('mousemove', resetTimer);
		window.removeEventListener('touchstart', resetTimer);
		window.removeEventListener('touchend', resetTimer);
	}
});

async function onClick(ev: MouseEvent, file: Misskey.entities.DriveFile) {
	const isShow = showingFiles.value.includes(file.id);
	if (!isShow) {
		ev.stopPropagation();
		if (file.isSensitive && !await confirmR18()) return;
		if (file.isSensitive && defaultStore.state.confirmWhenRevealingSensitiveMedia) {
			const { canceled } = await os.confirm({
				type: 'question',
				text: i18n.ts.sensitiveMediaRevealConfirm,
			});
			if (canceled) return;
			showingFiles.value.push(file.id);
		}
	}

	if (defaultStore.state.nsfwOpenBehavior === 'doubleClick') os.popup(MkRippleEffect, { x: ev.clientX, y: ev.clientY }, {});
	if (!isShow && defaultStore.state.nsfwOpenBehavior === 'click') {
		showingFiles.value.push(file.id);
	}
}

async function onDblClick(file: Misskey.entities.DriveFile) {
	if (file.isSensitive && !await confirmR18()) return;
	if (!showingFiles.value.includes(file.id) && defaultStore.state.nsfwOpenBehavior === 'doubleClick') {
		showingFiles.value.push(file.id);
	};
}
</script>

<style lang="scss" module>
.root {
	padding: 8px;
}

.stream {
	display: grid;
	grid-template-columns: repeat(auto-fill, minmax(160px, 1fr));
	grid-gap: 6px;
}

.img {
	position: relative;
	height: 128px;
	border-radius: 6px;
	overflow: clip;
}

.empty {
	margin: 0;
	padding: 16px;
	text-align: center;
}

.sensitiveImg {
	position: absolute;
	top: 0;
	left: 0;
	width: 100%;
	height: 100%;
	filter: brightness(0.7);
}
.hiddenTextWrapper {
	position: absolute;
	top: 0;
	left: 0;
	width: 100%;
	height: 100%;
	display: flex;
	text-align: center;
	align-items: center;
	justify-content: center;
	font-size: 0.8em;
	color: #fff;
	cursor: pointer;
}
</style>
