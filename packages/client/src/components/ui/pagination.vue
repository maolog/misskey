<template>
<transition name="fade" mode="out-in">
	<MkLoading v-if="fetching"/>

	<MkError v-else-if="error" @retry="init()"/>

	<div v-else-if="empty" key="_empty_" class="empty">
		<slot name="empty"></slot>
	</div>

	<div v-else class="cxiknjgy">
		<slot :items="items"></slot>
		<div v-show="more" key="_more_" class="more _gap">
			<MkButton v-appear="($store.state.enableInfiniteScroll && !disableAutoLoad) ? fetchMore : null" class="button" :disabled="moreFetching" :style="{ cursor: moreFetching ? 'wait' : 'pointer' }" primary @click="fetchMore">
				<template v-if="!moreFetching">{{ $ts.loadMore }}</template>
				<template v-if="moreFetching"><MkLoading inline/></template>
			</MkButton>
		</div>
	</div>
</transition>
</template>

<script lang="ts">
import { defineComponent } from 'vue';
import MkButton from './button.vue';
import paging from '@/scripts/paging';

export default defineComponent({
	components: {
		MkButton
	},

	mixins: [
		paging({}),
	],

	props: {
		pagination: {
			required: true
		},

		disableAutoLoad: {
			type: Boolean,
			required: false,
			default: false,
		}
	},
});
</script>

<style lang="scss" scoped>
.fade-enter-active,
.fade-leave-active {
	transition: opacity 0.125s ease;
}
.fade-enter-from,
.fade-leave-to {
	opacity: 0;
}

.cxiknjgy {
	> .more > .button {
		margin-left: auto;
		margin-right: auto;
		height: 48px;
		min-width: 150px;
	}
}
</style>
