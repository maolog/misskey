<template>
<div class="fzenkabp _debobigegoItem">
	<div class="_debobigegoPanel" :class="{ warn }">
		<i v-if="warn" class="fas fa-exclamation-triangle"></i>
		<i v-else class="fas fa-info-circle"></i>
		<slot></slot>
	</div>
</div>
</template>

<script lang="ts">
import { defineComponent } from 'vue';

export default defineComponent({
	props: {
		warn: {
			type: Boolean,
			required: false,
			default: false
		},
	},
	data() {
		return {
		};
	}
});
</script>

<style lang="scss" scoped>
.fzenkabp {
	> div {
		padding: 14px 16px;
		font-size: 90%;
		background: var(--infoBg);
		color: var(--infoFg);

		&.warn {
			background: var(--infoWarnBg);
			color: var(--infoWarnFg);
		}

		> i {
			margin-right: 4px;
		}
	}
}
</style>
