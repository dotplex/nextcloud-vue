<!--
 - @copyright Copyright (c) 2019 Christoph Wurst <christoph@winzerhof-wurst.at>
 -
 - @author Christoph Wurst <christoph@winzerhof-wurst.at>
 - @author Marco Ambrosini <marcoambrosini@pm.me>
 - @author John Molakvoæ <skjnldsv@protonmail.com>
 -
 - @license GNU AGPL version 3 or any later version
 -
 - This program is free software: you can redistribute it and/or modify
 - it under the terms of the GNU Affero General Public License as
 - published by the Free Software Foundation, either version 3 of the
 - License, or (at your option) any later version.
 -
 - This program is distributed in the hope that it will be useful,
 - but WITHOUT ANY WARRANTY; without even the implied warranty of
 - MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
 - GNU Affero General Public License for more details.
 -
 - You should have received a copy of the GNU Affero General Public License
 - along with this program. If not, see <http://www.gnu.org/licenses/>.
 -
 -->
<docs>
### General description

This components provides a wrapper around the main app's content.

Single-column layouts can just use the default slot. A resizable column
can be added by providing content to the named slot `list`.

### Examples

#### Usage: Single-column content
```vue
<template>
	<AppContent>
		<h2>Single-column main content</h2>
	</AppContent>
</template>
```

#### Usage: Two resizable columns
```vue
<template>
	<AppContent>
		<template slot="list">
			<div>Resizable list content</div>
		</template>

		<div>Main content</div>
	</AppContent>
</template>
```

#### Overriding Defaults
The default, min and max sizes (in percent) of the resizable list column can be overridden.
The list size must be between the min and the max width value.

```
<AppContent
	list-size="35"
	list-min-width="20"
	list-max-width="45"
>...</AppContent>
```
</docs>

<template>
	<main id="app-content-vue" class="app-content no-snapper">
		<div id="app-content-wrapper" :style="`height: ${contentHeight}px;`">
			<Splitpanes
				v-if="hasList"
				class="default-theme"
				@resized="handlePaneResize">
				<Pane class="list"
					:size="paneSizes.list || paneDefaults.list.size"
					:min-size="paneDefaults.list.min"
					:max-size="paneDefaults.list.max">
					<slot name="list" />
				</Pane>
				<Pane class="details"
					:size="paneSizes.details || paneDefaults.details.size"
					:min-size="paneDefaults.details.min"
					:max-size="paneDefaults.details.max">
					<slot />
				</Pane>
			</Splitpanes>
			<slot v-else />
		</div>
	</main>
</template>

<script>
import Hammer from 'hammerjs'
import { emit } from '@nextcloud/event-bus'
import { getBuilder } from '@nextcloud/browser-storage'
import { Splitpanes, Pane } from 'splitpanes'
import 'splitpanes/dist/splitpanes.css'

const browserStorage = getBuilder('nextcloud').persist().build()

/**
 * App content container to be used for the main content of your app
 *
 */
export default {
	name: 'AppContent',

	components: {
		Splitpanes,
		Pane,
	},

	props: {
		/**
		 * Allows to disable the control by swipe of the app navigation open state
		 */
		allowSwipeNavigation: {
			type: Boolean,
			default: true,
		},
		/**
		 * Allows you to set the default width of the resizable list in %
		 * Must be between listMinWidth and listMaxWidth
		 */
		listSize: {
			type: Number,
			required: false,
			default: 25,
		},
		/**
		 * Allows you to set the minimum width of the list column in %
		 */
		listMinWidth: {
			type: Number,
			required: false,
			default: 15,
		},
		/**
		 * Allows you to set the maximum width of the list column in %
		 */
		listMaxWidth: {
			type: Number,
			required: false,
			default: 40,
		},
	},

	data() {
		return {
			paneIdentifier: `pane-sizes.${this.$parent.$options.name}`,
			paneDefaults: {
				list: {},
				details: {},
			},
			contentHeight: 0,
		}
	},

	computed: {
		paneSizes() {
			const clientPaneSizes = browserStorage.getItem(this.paneIdentifier)
			return clientPaneSizes ? JSON.parse(clientPaneSizes) : {}
		},
		hasList() {
			return !!this.$slots.list
		},
	},

	mounted() {
		if (this.allowSwipeNavigation) {
			this.mc = new Hammer(this.$el, { cssProps: { userSelect: 'text' } })
			this.mc.on('swipeleft swiperight', this.handleSwipe)
		}
		this.setPaneDefaults()
		this.setContentHeight()
		window.addEventListener('resize', this.handleWindowResize)
	},
	beforeDestroy() {
		this.mc.off('swipeleft swiperight', this.handleSwipe)
		window.removeEventListener('resize', this.handleWindowResize)
	},
	methods: {
		// handle the swipe event
		handleSwipe(e) {
			const minSwipeX = 70
			const touchzone = 40
			const startX = e.srcEvent.pageX - e.deltaX
			const hasEnoughDistance = Math.abs(e.deltaX) > minSwipeX
			if (hasEnoughDistance && startX < touchzone) {
				emit('toggle-navigation', {
					open: true,
				})
			} else if (hasEnoughDistance && startX < touchzone + 300) {
				emit('toggle-navigation', {
					open: false,
				})
			}
		},
		handlePaneResize(event) {
			const paneSizes = {
				list: event[0].size.toFixed(2),
				details: event[1].size.toFixed(2),
			}
			browserStorage.setItem(this.paneIdentifier, JSON.stringify(paneSizes))
		},
		setContentHeight() {
			const header = document.getElementById('header')
			const headerHeight = header ? header.offsetHeight : 0
			this.contentHeight = window.innerHeight - headerHeight.offsetHeight
		},
		handleWindowResize() {
			this.setContentHeight()
		},
		setPaneDefaults() {
			this.paneDefaults.list.size = this.listSize
			this.paneDefaults.list.min = this.listMinWidth
			this.paneDefaults.list.max = this.listMaxWidth

			// set the inverse values of the details column
			// based on the provided (or default) values of the list column
			this.paneDefaults.details.size = 100 - this.listSize
			this.paneDefaults.details.min = 100 - this.listMaxWidth
			this.paneDefaults.details.max = 100 - this.listMinWidth
		},
	},
}
</script>
<style lang="scss" scoped>

.app-content {
	position: relative;
	z-index: 1000;
	flex-basis: 100vw;
	min-width: 0;
	min-height: 100%;
	// Overriding server styles TODO: cleanup!
	margin: 0 !important;
	background-color: var(--color-main-background);
}

#app-content-wrapper {
	overflow: hidden;
	height: inherit;
}

::v-deep .splitpanes.default-theme {
	.app-content-list {
		max-width: none;
	}

	.splitpanes__pane {
		background-color: transparent;
		transition: none;

		&.list {
			min-width: 200px;

			@media only screen and (max-width: 1024px) {
				display: none;
			}
		}

		&.details {
			overflow-y: scroll;

			@media only screen and (max-width: 1024px) {
				min-width: 100%;
			}
		}
	}

	.splitpanes__splitter {
		width: 9px;
		margin-left: -5px;
		background-color: transparent;
		border-left: none;

		&:before,
		&:after {
			display: none;
		}
	}
}
</style>
