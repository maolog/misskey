<template>
<div class="t9makv94">
	<section class="_section">
		<div class="_content">
			<details>
				<summary>{{ $ts.import }}</summary>
				<MkTextarea v-model="themeToImport">
					{{ $ts._theme.importInfo }}
				</MkTextarea>
				<MkButton :disabled="!themeToImport.trim()" @click="importTheme">{{ $ts.import }}</MkButton>
			</details>
		</div>
	</section>
	<section class="_section">
		<div class="_content _card _gap">
			<div class="_content">
				<MkInput v-model="name" required><span>{{ $ts.name }}</span></MkInput>
				<MkInput v-model="author" required><span>{{ $ts.author }}</span></MkInput>
				<MkTextarea v-model="description"><span>{{ $ts.description }}</span></MkTextarea>
				<div class="_inputs">
					<div v-text="$ts._theme.base" />
					<MkRadio v-model="baseTheme" value="light">{{ $ts.light }}</MkRadio>
					<MkRadio v-model="baseTheme" value="dark">{{ $ts.dark }}</MkRadio>
				</div>
			</div>
		</div>
		<div class="_content _card _gap">
			<div class="list-view _content">
				<div v-for="([ k, v ], i) in theme" :key="k" class="item">
					<div class="_inputs">
						<div>
							{{ k.startsWith('$') ? `${k} (${$ts._theme.constant})` : $t('_theme.keys.' + k) }}
							<button v-if="k.startsWith('$')" class="_button _link" @click="del(i)" v-text="$ts.delete" />
						</div>
						<div>
							<div class="type" @click="chooseType($event, i)">
								{{ getTypeOf(v) }} <i class="fas fa-chevron-down"></i>
							</div>
							<!-- default -->
							<div v-if="v === null" class="default-value" v-text="baseProps[k]" />
							<!-- color -->
							<div v-else-if="typeof v === 'string'" class="color">
								<input type="color" :value="v" @input="colorChanged($event.target.value, i)"/>
								<MkInput class="select" :value="v" @update:modelValue="colorChanged($event, i)"/>
							</div>
							<!-- ref const -->
							<MkInput v-else-if="v.type === 'refConst'" v-model="v.key">
								<template #prefix>$</template>
								<span>{{ $ts.name }}</span>
							</MkInput>
							<!-- ref props -->
							<MkSelect v-else-if="v.type === 'refProp'" v-model="v.key" class="select">
								<option v-for="key in themeProps" :key="key" :value="key">{{ $t('_theme.keys.' + key) }}</option>
							</MkSelect>
							<!-- func -->
							<template v-else-if="v.type === 'func'">
								<MkSelect v-model="v.name" class="select">
									<template #label>{{ $ts._theme.funcKind }}</template>
									<option v-for="n in ['alpha', 'darken', 'lighten']" :key="n" :value="n">{{ $t('_theme.' + n) }}</option>
								</MkSelect>
								<MkInput v-model="v.arg" type="number"><span>{{ $ts._theme.argument }}</span></MkInput>
								<MkSelect v-model="v.value" class="select">
									<template #label>{{ $ts._theme.basedProp }}</template>
									<option v-for="key in themeProps" :key="key" :value="key">{{ $t('_theme.keys.' + key) }}</option>
								</MkSelect>
							</template>
							<!-- CSS -->
							<MkInput v-else-if="v.type === 'css'" v-model="v.value">
								<span>CSS</span>
							</MkInput>
						</div>
					</div>
				</div>
				<MkButton primary @click="addConst">{{ $ts._theme.addConstant }}</MkButton>
			</div>
		</div>
	</section>
	<section class="_section">
		<details class="_content">
			<summary>{{ $ts.sample }}</summary>
			<MkSample/>
		</details>
	</section>
	<section class="_section">
		<div class="_content">
			<MkButton inline @click="preview">{{ $ts.preview }}</MkButton>
			<MkButton inline primary :disabled="!name || !author" @click="save">{{ $ts.save }}</MkButton>
		</div>
	</section>
</div>
</template>

<script lang="ts">
import { defineComponent } from 'vue';
import * as JSON5 from 'json5';
import { toUnicode } from 'punycode/';

import MkRadio from '@/components/form/radio.vue';
import MkButton from '@/components/ui/button.vue';
import MkInput from '@/components/form/input.vue';
import MkTextarea from '@/components/form/textarea.vue';
import MkSelect from '@/components/form/select.vue';
import MkSample from '@/components/sample.vue';

import { convertToMisskeyTheme, ThemeValue, convertToViewModel, ThemeViewModel } from '@/scripts/theme-editor';
import { Theme, applyTheme, lightTheme, darkTheme, themeProps, validateTheme } from '@/scripts/theme';
import { host } from '@/config';
import * as os from '@/os';
import { ColdDeviceStorage } from '@/store';
import { addTheme } from '@/theme-store';
import * as symbols from '@/symbols';

export default defineComponent({
	components: {
		MkRadio,
		MkButton,
		MkInput,
		MkTextarea,
		MkSelect,
		MkSample,
	},

	async beforeRouteLeave(to, from, next) {
		if (this.changed && !(await this.confirm())) {
			next(false);
		} else {
			next();
		}
	},

	data() {
		return {
			[symbols.PAGE_INFO]: {
				title: this.$ts.themeEditor,
				icon: 'fas fa-palette',
			},
			theme: [] as ThemeViewModel,
			name: '',
			description: '',
			baseTheme: 'light' as 'dark' | 'light',
			author: `@${this.$i.username}@${toUnicode(host)}`,
			themeToImport: '',
			changed: false,
			lightTheme, darkTheme, themeProps,
		}
	},

	computed: {
		baseProps() {
			return this.baseTheme === 'light' ? this.lightTheme.props : this.darkTheme.props;
		},
	},

	beforeUnmount() {
		window.removeEventListener('beforeunload', this.beforeunload);
	},

	mounted() {
		this.init();
		window.addEventListener('beforeunload', this.beforeunload);
		const changed = () => this.changed = true;
		this.$watch('name', changed);
		this.$watch('description', changed);
		this.$watch('baseTheme', changed);
		this.$watch('author', changed);
		this.$watch('theme', changed);
	},

	methods: {
		beforeunload(e: BeforeUnloadEvent) {
			if (this.changed) {
				e.preventDefault();
				e.returnValue = '';
			}
		},

		async confirm(): Promise<boolean> {
			const { canceled } = await os.confirm({
				type: 'warning',
				text: this.$ts.leaveConfirm,
			});
			return !canceled;
		},

		init() {
			const t: ThemeViewModel = [];
			for (const key of themeProps) {
				t.push([ key, null ]);
			}
			this.theme = t;
		},
	
		async del(i: number) {
			const { canceled } = await os.confirm({ 
				type: 'warning',
				text: this.$t('_theme.deleteConstantConfirm', { const: this.theme[i][0] }),
			});
			if (canceled) return;
			Vue.delete(this.theme, i);
		},
	
		async addConst() {
			const { canceled, result } = await os.inputText({
				title: this.$ts._theme.inputConstantName,
			});
			if (canceled) return;
			this.theme.push([ '$' + result, '#000000']);
		},
	
		save() {
			const theme = convertToMisskeyTheme(this.theme, this.name, this.description, this.author, this.baseTheme);
			addTheme(theme);
			os.alert({
				type: 'success',
				text: this.$t('_theme.installed', { name: theme.name })
			});
			this.changed = false;
		},
	
		preview() {
			const theme = convertToMisskeyTheme(this.theme, this.name, this.description, this.author, this.baseTheme);
			try {
				applyTheme(theme, false);
			} catch (e) {
				os.alert({
					type: 'error',
					text: e.message
				});
			}
		},
	
		async importTheme() {
			if (this.changed && (!await this.confirm())) return;

			try {
				const theme = JSON5.parse(this.themeToImport) as Theme;
				if (!validateTheme(theme)) throw new Error(this.$ts._theme.invalid);

				this.name = theme.name;
				this.description = theme.desc || '';
				this.author = theme.author;
				this.baseTheme = theme.base || 'light';
				this.theme = convertToViewModel(theme);
				this.themeToImport = '';
			} catch (e) {
				os.alert({
					type: 'error',
					text: e.message
				});
			}
		},
	
		colorChanged(color: string, i: number) {
			this.theme[i] = [this.theme[i][0], color];
		},

		getTypeOf(v: ThemeValue) {
			return v === null
				? this.$ts._theme.defaultValue
				: typeof v === 'string'
					? this.$ts._theme.color
					: this.$t('_theme.' + v.type);
		},
	
		async chooseType(e: MouseEvent, i: number) {
			const newValue = await this.showTypeMenu(e);
			this.theme[i] = [ this.theme[i][0], newValue ];
		},
	
		showTypeMenu(e: MouseEvent) {
			return new Promise<ThemeValue>((resolve) => {
				os.popupMenu([{
					text: this.$ts._theme.defaultValue,
					action: () => resolve(null),
				}, {
					text: this.$ts._theme.color,
					action: () => resolve('#000000'),
				}, {
					text: this.$ts._theme.func,
					action: () => resolve({
						type: 'func', name: 'alpha', arg: 1, value: 'accent'
					}),
				}, {
					text: this.$ts._theme.refProp,
					action: () => resolve({
						type: 'refProp', key: 'accent',
					}),
				}, {
					text: this.$ts._theme.refConst,
					action: () => resolve({
						type: 'refConst', key: '',
					}),
				}, {
					text: 'CSS',
					action: () => resolve({
						type: 'css', value: '',
					}),
				}], e.currentTarget || e.target);
			});
		}
	}
});
</script>

<style lang="scss" scoped>
.t9makv94 {
	> ._section {
		> ._content {
			> .list-view {
				> .item {
					min-height: 48px;
					word-break: break-all;

					&:not(:last-child) {
						margin-bottom: 8px;
					}

					.select {
						margin: 24px 0;
					}

					.type {
						cursor: pointer;
					}

					.default-value {
						opacity: 0.6;
						pointer-events: none;
						user-select: none;
					}

					.color {
						> input {
							display: inline-block;
							width: 1.5em;
							height: 1.5em;
						}

						> div {
							margin-left: 8px;
							display: inline-block;
						}
					}
				}
			}
		}
	}
}
</style>
