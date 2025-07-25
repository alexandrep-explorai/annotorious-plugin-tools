<script lang="ts">
  import { createEventDispatcher, onMount } from 'svelte';
  import { ShapeType } from '@annotorious/annotorious';
  import { getMaskDimensions } from '@annotorious/annotorious';
  import type { Ellipse, DrawingMode, Transform } from '@annotorious/annotorious';

  const dispatch = createEventDispatcher<{ create: Ellipse }>();
  
  /** Props **/
  export let addEventListener: (type: string, fn: EventListener, capture?: boolean) => void;
  export let drawingMode: DrawingMode
  export let transform: Transform;
  export let viewportScale: number;
  
  let container: SVGGElement;

  let origin: [x: number, y: number] | undefined; 

  let anchor: [number, number] | undefined;

  let x: number, y: number, w: number, h: number;

  let isShiftPressed = false;

  let isCtrlPressed = false;

  let lastPointerDown: number;

  let lastMoveEvent: PointerEvent | undefined;

  const onPointerDown = (event: Event) => {
    const evt = event as PointerEvent;

    lastPointerDown = performance.now();

    if (drawingMode === 'drag') {
      origin = transform.elementToImage(evt.offsetX, evt.offsetY);
      anchor = origin;

      x = origin[0];
      y = origin[1];
      w = 1;
      h = 1;
    }
  }

  const updateShape = (maybeEvent?: Event) => {
    const evt = (maybeEvent as PointerEvent)|| lastMoveEvent;

    if (origin) {
      anchor = transform.elementToImage(evt!.offsetX, evt!.offsetY);

      if (isCtrlPressed) {
        const mw = 2 * Math.abs(anchor[0] - origin[0]);
        const mh = 2 * Math.abs(anchor[1] - origin[1]);

        w = isShiftPressed ? Math.max(mw, mh) : mw;
        h = isShiftPressed ? w : mh;

        x = Math.min(anchor[0], origin[0] - w / 2);
        y = Math.min(anchor[1], origin[1] - h / 2);
      } else {
        const mw = Math.abs(anchor[0] - origin[0]);
        const mh = Math.abs(anchor[1] - origin[1]);

        w = isShiftPressed ? Math.max(mw, mh) : mw;
        h = isShiftPressed ? w : mh;

        x = Math.min(anchor[0], origin[0]);
        y = Math.min(anchor[1], origin[1]);
      }
    }

    if (maybeEvent)
      lastMoveEvent = maybeEvent as PointerEvent;
  }
    
  const onPointerUp = (event: Event) => {
    const evt = event as PointerEvent;
    
    const timeDifference = performance.now() - lastPointerDown;

    if (drawingMode === 'click') {
      // Not a single click - ignore
      if (timeDifference > 300)
        return;

      // This statement caused a weird bug on OSD: when starting
      // to draw with a quick drag (<300ms), OSD got stuck in mouseNav 
      // mode. The image still moved with the mouse cursor, even though 
      // button was no longer pressed. 
      // I'm commenting out this statement as a fix. But there must have
      // been a reason I put it here in the first place. Keep an eye ou
      // for regressions.
      // 
      // And if you are ever tempted to un-comment the statement: beware!
      
      // evt.stopPropagation();

      if (origin) {
        stopDrawing();
      } else {
        // Start drawing
        origin = transform.elementToImage(evt.offsetX, evt.offsetY);
        anchor = origin;

        x = origin[0];
        y = origin[1];
        w = 1;
        h = 1;
      }
    } else if (origin) {
      if (timeDifference > 300 || w * h > 100) {
        evt.stopPropagation();
        stopDrawing();
      } else {
        origin = undefined;
        anchor = undefined;

        lastMoveEvent = undefined;
      }
    }
  }

  const stopDrawing = () => {
    // Require 4x4 pixels minimum
    if (w * h > 15) {
      const shape: Ellipse = {
        type: ShapeType.ELLIPSE, 
        geometry: {
          bounds: {
            minX: x, 
            minY: y,
            maxX: x + w,
            maxY: y + h
          },
          cx: x + w / 2,
          cy: y + h / 2,
          rx: w / 2,
          ry: h / 2
        }
      }

      dispatch('create', shape);
    }

    origin = undefined;
    anchor = undefined;

    lastMoveEvent = undefined;
  }

  const onKeyDown = (evt: KeyboardEvent) => {
    if (evt.key === 'Shift') {
      isShiftPressed = true;  
      updateShape();  
    }

    if (evt.key === 'Control') {
      isCtrlPressed = true;
      updateShape();
    }
  }

  const onKeyUp = (evt: KeyboardEvent) => {
    if (evt.key === 'Shift') {
      isShiftPressed = false;
      updateShape();  
    }

    if (evt.key === 'Control') {
      isCtrlPressed = false;
      updateShape();  
    }
  }

  onMount(() => {
    document.addEventListener('keyup', onKeyUp);
    document.addEventListener('keydown', onKeyDown);

    addEventListener('pointerdown', onPointerDown);
    addEventListener('pointermove', updateShape);
    addEventListener('pointerup', onPointerUp, true);

    return () => {
      document.removeEventListener('keyup', onKeyUp);
      document.removeEventListener('keydown', onKeyDown);
    }
  });

  const maskId = `ellipse-mask-${Math.random().toString(36).substring(2, 12)}`;
  $: buffer = 2 / viewportScale;
</script>

<g 
  bind:this={container}
  class="a9s-annotation a9s-rubberband">
  
  {#if origin}
    <defs>
      <mask id={maskId} class="a9s-rubberband-ellipse-mask">
        <rect 
          x={x - buffer} 
          y={y - buffer} 
          width={w + 2 * buffer}
          height={h + 2 * buffer}/>

        <ellipse
          cx={x + w / 2} 
          cy={y + h / 2} 
          rx={w / 2} 
          ry={h / 2} />
      </mask>
    </defs>

    <ellipse 
      class="a9s-outer"
      mask={`url(#${maskId})`}
      cx={x + w / 2} 
      cy={y + h / 2} 
      rx={w / 2} 
      ry={h / 2} />

    <ellipse 
      class="a9s-inner"
      cx={x + w / 2} 
      cy={y + h / 2} 
      rx={w / 2} 
      ry={h / 2} />
  {/if}
</g>

<style>
  mask.a9s-rubberband-ellipse-mask > rect {
    fill: #fff;
  }

  mask.a9s-rubberband-ellipse-mask > ellipse {
    fill: #000;
  }
</style>