<script>
	import { setContext } from 'svelte';
	import { writable, derived } from 'svelte/store';

	import makeAccessor from './utils/makeAccessor.js';
	import filterObject from './utils/filterObject.js';
	import calcExtents from './lib/calcExtents.js';
	import calcDomain from './helpers/calcDomain.js';
	import createScale from './helpers/createScale.js';
	import createGetter from './helpers/createGetter.js';
	import getRange from './helpers/getRange.js';
	import defaultScales from './settings/defaultScales.js';
	import defaultReverses from './settings/defaultReverses.js';

	export let ssr = false;
	export let pointerEvents = true;
	export let position = 'relative';
	export let percentRange = false;

	export let width = undefined;
	export let height = undefined;

	export let containerWidth = width || 100;
	export let containerHeight = height || 100;

	export let element = undefined;

	/* --------------------------------------------
	 * Parameters
	 * Values that computed properties are based on and that
	 * can be easily extended from config values
	 *
	 */
	export let x = undefined;
	export let y = undefined;
	export let z = undefined;
	export let r = undefined;
	export let custom = {};
	export let data = [];
	export let xDomain = undefined;
	export let yDomain = undefined;
	export let zDomain = undefined;
	export let rDomain = undefined;
	export let xNice = false;
	export let yNice = false;
	export let zNice = false;
	export let rNice = false;
	export let xReverse = defaultReverses.x;
	export let yReverse = defaultReverses.y;
	export let zReverse = defaultReverses.z;
	export let rReverse = defaultReverses.r;
	export let xPadding = undefined;
	export let yPadding = undefined;
	export let zPadding = undefined;
	export let rPadding = undefined;
	export let xScale = defaultScales.x;
	export let yScale = defaultScales.y;
	export let zScale = defaultScales.y;
	export let rScale = defaultScales.r;
	export let xRange = undefined;
	export let yRange = undefined;
	export let zRange = undefined;
	export let rRange = undefined;
	export let padding = {};
	export let extents = {};
	export let flatData = undefined;

	/* --------------------------------------------
	 * Preserve a copy of our passed in settings before we modify them
	 * Return this to the user's context so they can reference things if need be
	 * Add the active keys since those aren't on our settings object.
	 * This is mostly an escape-hatch
	 */
	const config = {};
	$: if (x) config.x = x;
	$: if (y) config.y = y;
	$: if (z) config.z = z;
	$: if (r) config.r = r;
	$: if (xDomain) config.xDomain = xDomain;
	$: if (yDomain) config.yDomain = yDomain;
	$: if (zDomain) config.zDomain = zDomain;
	$: if (rDomain) config.rDomain = rDomain;
	$: if (xRange) config.xRange = xRange;
	$: if (yRange) config.yRange = yRange;
	$: if (zRange) config.zRange = zRange;
	$: if (rRange) config.rRange = rRange;

	/* --------------------------------------------
	 * Make store versions of each parameter
	 * Prefix these with `_` to keep things organized
	 */
	const _percentRange = writable();
	const _containerWidth = writable();
	const _containerHeight = writable();
	const _x = writable();
	const _y = writable();
	const _z = writable();
	const _r = writable();
	const _custom = writable();
	const _data = writable();
	const _xDomain = writable();
	const _yDomain = writable();
	const _zDomain = writable();
	const _rDomain = writable();
	const _xNice = writable();
	const _yNice = writable();
	const _zNice = writable();
	const _rNice = writable();
	const _xReverse = writable();
	const _yReverse = writable();
	const _zReverse = writable();
	const _rReverse = writable();
	const _xPadding = writable();
	const _yPadding = writable();
	const _zPadding = writable();
	const _rPadding = writable();
	const _xScale = writable();
	const _yScale = writable();
	const _zScale = writable();
	const _rScale = writable();
	const _xRange = writable();
	const _yRange = writable();
	const _zRange = writable();
	const _rRange = writable();
	const _padding = writable();
	const _flatData = writable();
	const _extents = writable();
	const _config = writable(config);

	$: _percentRange.set(percentRange);
	$: _containerWidth.set(containerWidth);
	$: _containerHeight.set(containerHeight);
	$: _x.set(makeAccessor(x));
	$: _y.set(makeAccessor(y));
	$: _z.set(makeAccessor(z));
	$: _r.set(makeAccessor(r));
	$: _xDomain.set(xDomain);
	$: _yDomain.set(yDomain);
	$: _zDomain.set(zDomain);
	$: _rDomain.set(rDomain);
	$: _custom.set(custom);
	$: _data.set(data);
	$: _xNice.set(xNice);
	$: _yNice.set(yNice);
	$: _zNice.set(zNice);
	$: _rNice.set(rNice);
	$: _xReverse.set(xReverse);
	$: _yReverse.set(yReverse);
	$: _zReverse.set(zReverse);
	$: _rReverse.set(rReverse);
	$: _xPadding.set(xPadding);
	$: _yPadding.set(yPadding);
	$: _zPadding.set(zPadding);
	$: _rPadding.set(rPadding);
	$: _xScale.set(xScale);
	$: _yScale.set(yScale);
	$: _zScale.set(zScale);
	$: _rScale.set(rScale);
	$: _xRange.set(xRange);
	$: _yRange.set(yRange);
	$: _zRange.set(zRange);
	$: _rRange.set(rRange);
	$: _padding.set(padding);
	$: _extents.set(filterObject(extents));
	$: _flatData.set(flatData || data);

	/* --------------------------------------------
	 * Create derived values
	 * Suffix these with `_d`
	 */
	const activeGetters_d = derived([_x, _y, _z, _r], ([$x, $y, $z, $r]) => {
		const obj = {};
		if ($x) {
			obj.x = $x;
		}
		if ($y) {
			obj.y = $y;
		}
		if ($z) {
			obj.z = $z;
		}
		if ($r) {
			obj.r = $r;
		}
		return obj;
	});

	const padding_d = derived([_padding, _containerWidth, _containerHeight], ([$padding]) => {
		const defaultPadding = { top: 0, right: 0, bottom: 0, left: 0 };
		return Object.assign(defaultPadding, $padding);
	});

	const box_d = derived([_containerWidth, _containerHeight, padding_d], ([$containerWidth, $containerHeight, $padding]) => {
		const b = {};
		b.top = $padding.top;
		b.right = $containerWidth - $padding.right;
		b.bottom = $containerHeight - $padding.bottom;
		b.left = $padding.left;
		b.width = b.right - b.left;
		b.height = b.bottom - b.top;
		if (b.width <= 0) {
			console.error('[LayerCake] Target div has zero or negative width. Did you forget to set an explicit width in CSS on the container?');
		}
		if (b.height <= 0) {
			console.error('[LayerCake] Target div has zero or negative height. Did you forget to set an explicit height in CSS on the container?');
		}
		return b;
	});

	const width_d = derived([box_d], ([$box]) => {
		return $box.width;
	});

	const height_d = derived([box_d], ([$box]) => {
		return $box.height;
	});

	/* --------------------------------------------
	 * Calculate extents by taking the extent of the data
	 * and filling that in with anything set by the user
	 */
	const extents_d = derived([_flatData, activeGetters_d, _extents], ([$flatData, $activeGetters, $extents]) => {
		return { ...calcExtents($flatData, filterObject($activeGetters, $extents)), ...$extents };
	});

	const xDomain_d = derived([extents_d, _xDomain], calcDomain('x'));
	const yDomain_d = derived([extents_d, _yDomain], calcDomain('y'));
	const zDomain_d = derived([extents_d, _zDomain], calcDomain('z'));
	const rDomain_d = derived([extents_d, _rDomain], calcDomain('r'));

	const xScale_d = derived([_xScale, extents_d, xDomain_d, _xPadding, _xNice, _xReverse, width_d, height_d, _xRange, _percentRange], createScale('x'));
	const xGet_d = derived([_x, xScale_d], createGetter);

	const yScale_d = derived([_yScale, extents_d, yDomain_d, _yPadding, _yNice, _yReverse, width_d, height_d, _yRange, _percentRange], createScale('y'));
	const yGet_d = derived([_y, yScale_d], createGetter);

	const zScale_d = derived([_zScale, extents_d, zDomain_d, _zPadding, _zNice, _zReverse, width_d, height_d, _zRange, _percentRange], createScale('z'));
	const zGet_d = derived([_z, zScale_d], createGetter);

	const rScale_d = derived([_rScale, extents_d, rDomain_d, _rPadding, _rNice, _rReverse, width_d, height_d, _rRange, _percentRange], createScale('r'));
	const rGet_d = derived([_r, rScale_d], createGetter);

	const xRange_d = derived([xScale_d], getRange);
	const yRange_d = derived([yScale_d], getRange);
	const zRange_d = derived([zScale_d], getRange);
	const rRange_d = derived([rScale_d], getRange);

	const aspectRatio_d = derived([width_d, height_d], ([$aspectRatio, $width, $height]) => {
		return $width / $height;
	});

	$: context = {
		activeGetters: activeGetters_d,
		width: width_d,
		height: height_d,
		percentRange: _percentRange,
		aspectRatio: aspectRatio_d,
		containerWidth: _containerWidth,
		containerHeight: _containerHeight,
		x: _x,
		y: _y,
		z: _z,
		r: _r,
		custom: _custom,
		data: _data,
		xNice: _xNice,
		yNice: _yNice,
		zNice: _zNice,
		rNice: _rNice,
		xReverse: _xReverse,
		yReverse: _yReverse,
		zReverse: _zReverse,
		rReverse: _rReverse,
		xPadding: _xPadding,
		yPadding: _yPadding,
		zPadding: _zPadding,
		rPadding: _rPadding,
		padding: padding_d,
		flatData: _flatData,
		extents: extents_d,
		xDomain: xDomain_d,
		yDomain: yDomain_d,
		zDomain: zDomain_d,
		rDomain: rDomain_d,
		xRange: xRange_d,
		yRange: yRange_d,
		zRange: zRange_d,
		rRange: rRange_d,
		config: _config,
		xScale: xScale_d,
		xGet: xGet_d,
		yScale: yScale_d,
		yGet: yGet_d,
		zScale: zScale_d,
		zGet: zGet_d,
		rScale: rScale_d,
		rGet: rGet_d
	};

	$: setContext('LayerCake', context);
</script>

{#if (ssr === true || typeof window !== 'undefined')}
	<div
		bind:this={element}
		class="layercake-container"
		style="
			position:{position};
			{position === 'absolute' ? 'top:0;right:0;bottom:0;left:0;' : ''}
			{pointerEvents === false ? 'pointer-events:none;' : ''}
		"
		bind:clientWidth={containerWidth}
		bind:clientHeight={containerHeight}
	>
		<slot
			{element}
			width={$width_d}
			height={$height_d}
			aspectRatio={$aspectRatio_d}
			containerWidth={$_containerWidth}
			containerHeight={$_containerHeight}
		></slot>
	</div>
{/if}

<style>
	.layercake-container,
	.layercake-container :global(*) {
		box-sizing: border-box;
	}
	.layercake-container {
		width: 100%;
		height: 100%;
	}
</style>
