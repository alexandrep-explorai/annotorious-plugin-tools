<script lang="ts">
  import { createEventDispatcher, onMount, tick } from 'svelte';
  import { Editor, Handle, MidpointHandle } from '@annotorious/annotorious/src';
  import { boundsFromPoints, getMaskDimensions, isTouch } from '@annotorious/annotorious';
  import type { Line, LineGeometry, Shape, Transform } from '@annotorious/annotorious';

  const dispatch = createEventDispatcher<{ change: Line }>();
  
  /** Time difference (milliseconds) required for registering a click/tap **/
  const CLICK_THRESHOLD = 250;

  /** Maximum time difference (milliseconds) for registering double click */
  const DOUBLE_CLICK_THRESHOLD = 500;

  /** Minimum distance (px) to shape required for midpoints to show */
  const MIN_HOVER_DISTANCE = 1000;

  /** Minimum distance (px) between corners required for midpoints to show **/
  const MIN_CORNER_DISTANCE = 12;

  /** Needed for the <mask> element **/
  const MIDPOINT_SIZE = 4.5;

  /** Props */
  export let shape: Line;
  export let computedStyle: string | undefined;
  export let transform: Transform;
  export let viewportScale: number = 1;
  export let svgEl: SVGSVGElement;

  /** Drawing tool layer **/
  let visibleMidpoint: number | undefined;
  let isHandleHovered = false;
  let lastHandleClick: number | undefined;
  let selectedCorners: number[] = [];

  $: geom = shape.geometry;

  // No support yet for adding or removing points in mobile!
  $: midpoints = isTouch ? [] : geom.points.reduce((acc, thisCorner, idx) => {
    if (idx === geom.points.length - 1) return acc;
    const nextCorner = geom.points[idx + 1];
    
    const x = (thisCorner[0] + nextCorner[0]) / 2;
    const y = (thisCorner[1] + nextCorner[1]) / 2;

    const dist = Math.sqrt( 
      Math.pow(nextCorner[0] - x, 2) + Math.pow(nextCorner[1] - y, 2));

    // Don't show if the distance between the corners is too small
    const visible = dist > MIN_CORNER_DISTANCE / viewportScale;

    return [...acc, { point: [x, y], visible }];
  }, [] as { point: number[], visible: boolean }[]);

  /** Handle hover state **/
  const onEnterHandle = () => isHandleHovered = true;
  const onLeaveHandle = () => isHandleHovered = false;

  /** Determine visible midpoint, if any **/
  const onPointerMove = (evt: PointerEvent) => {
    if (selectedCorners.length > 0 || !midpoints.some(m => m.visible)) {
      visibleMidpoint = undefined;
      return;
    }
    
    const [px, py] = transform.elementToImage(evt.offsetX, evt.offsetY);

    const getDistSq = (pt: number[]) =>
      Math.pow(pt[0] - px, 2) + Math.pow(pt[1] - py, 2);

    const closestCorner = geom.points.reduce((closest, corner) =>
      getDistSq(corner) < getDistSq(closest) ? corner : closest);

    const closestVisibleMidpoint = midpoints
      .filter(m => m.visible)
      .reduce((closest, midpoint) =>
        getDistSq(midpoint.point) < getDistSq(closest.point) ? midpoint : closest);

    // Show midpoint of the mouse is at least within THRESHOLD distance
    // of the midpoint or the closest corner. (Basically a poor man's shape buffering).
    const threshold = Math.pow(MIN_HOVER_DISTANCE / viewportScale, 2);

    const shouldShow = 
      getDistSq(closestCorner) < threshold ||
      getDistSq(closestVisibleMidpoint.point) < threshold;

    if (shouldShow)
      visibleMidpoint = midpoints.indexOf(closestVisibleMidpoint);
    else
      visibleMidpoint = undefined;
  }

  /** 
   * SVG element keeps loosing focus when interacting with 
   * shapes–this function refocuses.
   */
   const reclaimFocus = () => {
    if (document.activeElement !== svgEl)
      svgEl.focus();
  }

  /**
   * De-selects all corners and reclaims focus.
   */
  const onLinePointerUp = () => {
    selectedCorners = [];
    reclaimFocus();
  }

  /**
   * Updates state, waiting for potential click.
   */
  const onHandlePointerDown = (evt: PointerEvent) => {
    isHandleHovered = true;

    evt.preventDefault();
    evt.stopPropagation();

    lastHandleClick = performance.now();
  }

  /** Selection handling logic **/
  const onHandlePointerUp = (idx: number) => (evt: PointerEvent) => {
    if (!lastHandleClick || isTouch) return;

    // Drag, not click
    if (performance.now() - lastHandleClick > CLICK_THRESHOLD) return;
    
    const isSelected = selectedCorners.includes(idx);

    if (evt.metaKey || evt.ctrlKey || evt.shiftKey) {
      // Add to or remove from selection
      if (isSelected)
        selectedCorners = selectedCorners.filter(i => i !== idx);
      else
        selectedCorners = [...selectedCorners, idx];
    } else {
      if (isSelected && selectedCorners.length > 1)
        // Keep selected, de-select others
        selectedCorners = [idx]
      else if (isSelected)
        // De-select
        selectedCorners = [];
      else
        selectedCorners = [idx];
    }

    reclaimFocus();
  }

  const editor = (line: Shape, handle: string, delta: [number, number]) => {
    let points: [number, number][];

    const geom = line.geometry as LineGeometry;
    const [dx, dy] = delta;

    if (selectedCorners.length > 1) {
      points = geom.points.map(([x, y], idx) =>
        selectedCorners.includes(idx) ? [x + dx, y + dy] : [x, y]);
    } else if (handle === 'LINE') {
      points = geom.points.map(([x, y]) => [x + dx, y + dy]);
    } else {
      points = geom.points.map(([x, y], idx) =>
        handle === `HANDLE-${idx}` ? [x + dx, y + dy] : [x, y]);
    }

    const bounds = boundsFromPoints(points);
    return {
      ...line,
      geometry: { bounds, points }
    };
  }

  const onAddPoint = (insertionIndex: number, point?: number[]) => async (evt: PointerEvent) => {
    evt.stopPropagation();

    const points = [
      ...geom.points.slice(0, insertionIndex + 1),
      midpoints[insertionIndex]?.point ?? point,
      ...geom.points.slice(insertionIndex + 1)
    ] as [number, number][];

    const bounds = boundsFromPoints(points);

    dispatch('change', {
      ...shape,
      geometry: { points, bounds }
    });

    await tick();

    // Find the newly inserted handle and dispatch grab event
    const newHandle = [...document.querySelectorAll(`.a9s-handle`)][insertionIndex + 1];
    if (newHandle?.firstChild) {
      const newEvent = new PointerEvent('pointerdown', {
        bubbles: true,
        cancelable: true,
        clientX: evt.clientX,
        clientY: evt.clientY,
        pointerId: evt.pointerId,
        pointerType: evt.pointerType,
        isPrimary: evt.isPrimary,
        buttons: evt.buttons
      });

      newHandle.firstChild.dispatchEvent(newEvent);
    }
  }

  const onDeleteSelected = () => {
    // Line needs 2 points min
    if (geom.points.length - selectedCorners.length < 2) return;

    const points = geom.points.filter((_, i) => !selectedCorners.includes(i)) as [number, number][];
    const bounds = boundsFromPoints(points);

    dispatch('change', {
      ...shape,
      geometry: { points, bounds }
    });

    selectedCorners = [];
  }

  onMount(() => {
    if (isTouch) return;

    const onKeydown = (evt: KeyboardEvent) => {
      if (evt.key === 'Delete' || evt.key === 'Backspace') {
        evt.preventDefault();
        onDeleteSelected();
      }
    };

    svgEl.addEventListener('pointermove', onPointerMove);
    svgEl.addEventListener('keydown', onKeydown);

    return () => {
      svgEl.removeEventListener('pointermove', onPointerMove);
      svgEl.removeEventListener('keydown', onKeydown);
    }
  });

  $: mask = getMaskDimensions(geom.bounds, MIDPOINT_SIZE / viewportScale);

  const maskId = `line-mask-${Math.random().toString(36).substring(2, 12)}`;

</script>

<Editor
  shape={shape}
  transform={transform}
  editor={editor}
  svgEl={svgEl}
  on:grab
  on:change
  on:release
  let:grab={grab}>

  <defs>
    {#if (visibleMidpoint !== undefined && !isHandleHovered)}
      {@const { point } = midpoints[visibleMidpoint]}
      <mask id={maskId}  class="a9s-line-editor-mask" maskUnits="userSpaceOnUse">
        <rect x={mask.x} y={mask.y} width={mask.w} height={mask.h} /> 
        <circle cx={point[0]} cy={point[1]} r={MIDPOINT_SIZE / viewportScale} />
      </mask>
    {/if}
  </defs>
  
  <polyline
    class="a9s-outer"
    mask={`url(#${maskId})`}
    on:pointerup={onLinePointerUp}
    on:pointerdown={grab('LINE')}
    points={geom.points.map(xy => xy.join(',')).join(' ')} />

  <polyline
    class="a9s-inner a9s-shape-handle"
    mask={`url(#${maskId})`}
    style={computedStyle}
    on:pointerup={onLinePointerUp}
    on:pointerdown={grab('LINE')}
    points={geom.points.map(xy => xy.join(',')).join(' ')} />

  {#each geom.points as point, idx}
    <Handle 
      class="a9s-corner-handle"
      x={point[0]}
      y={point[1]}
      scale={viewportScale}
      selected={selectedCorners.includes(idx)}
      on:pointerenter={onEnterHandle}
      on:pointerleave={onLeaveHandle}
      on:pointerdown={onHandlePointerDown}
      on:pointerdown={grab(`HANDLE-${idx}`)}
      on:pointerup={onHandlePointerUp(idx)} />
  {/each}

  {#if (visibleMidpoint !== undefined && !isHandleHovered)}
    {@const { point } = midpoints[visibleMidpoint]}
    <MidpointHandle 
      x={point[0]}
      y={point[1]}
      scale={viewportScale}
      on:pointerdown={onAddPoint(visibleMidpoint)} />
  {/if}

</Editor>

<style>
  mask.a9s-line-editor-mask > rect {
    fill: #fff;
  }

  mask.a9s-line-editor-mask > circle {
    fill: #000;
  }
</style>