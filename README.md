# ASCII Animation Tutorial

This tutorial explains how to create a smooth, efficient ASCII animation in Next.js using Ghostty's animation approach.

## Overview

We created a high-performance ASCII animation component that:
- Uses `requestAnimationFrame` for smooth 60fps rendering
- Loads ASCII frames from static files
- Applies color overlays with CSS blend modes
- Respects user motion preferences
- Pauses when window loses focus

## Prerequisites

To generate ASCII frames, you need **ImageMagick** installed on your machine:

- **macOS:**  
  ```bash
  brew install imagemagick
- **Linux:**  
  ```bash
  sudo apt install imagemagick
- **Windows:**  
You can download ImageMagick here: https://imagemagick.org/script/download.php

## Step 1: Convert Video to ASCII Frames

### 1.1 Create ASCII Conversion Script

Create `ascii.sh` with these key configurations:

```bash
# Configuration
FONT_RATIO="0.44"
LUMINANCE_THRESHOLD=15  # Adjust to remove more/less background
ASCII_CHARS=" .'\`^,:;Il!i><~+_-?][}{1)(|/tfjrxnuvczXYUJCLQ0OZmwqpdbkhao*#MW&8%B@$"
OUTPUT_FPS=24
OUTPUT_COLUMNS=100
```

### 1.2 Generate ASCII Frames

```bash
./ascii.sh your-video.mp4
```

This creates a timestamped folder with 60 `.txt` files containing ASCII art.

### 1.3 Move Frames to Public Directory

```bash
mkdir -p public/frames
cp ascii_frames_*/frame_images/frame_*.txt public/frames/
```

## Step 2: Create Animation Manager Class

### 2.1 RequestAnimationFrame-Based Timing

```typescript
class AnimationManager {
  private _animation: number | null = null;
  private callback: () => void;
  private lastFrame = -1;
  private frameTime = 1000 / 30; // 30fps

  constructor(callback: () => void, fps = 30) {
    this.callback = callback;
    this.frameTime = 1000 / fps;
  }

  start() {
    if (this._animation != null) return;
    this._animation = requestAnimationFrame(this.update);
  }

  pause() {
    if (this._animation == null) return;
    this.lastFrame = -1;
    cancelAnimationFrame(this._animation);
    this._animation = null;
  }

  private update = (time: number) => {
    const { lastFrame } = this;
    let delta = time - lastFrame;
    if (this.lastFrame === -1) {
      this.lastFrame = time;
    } else {
      while (delta >= this.frameTime) {
        this.callback();
        delta -= this.frameTime;
        this.lastFrame += this.frameTime;
      }
    }
    this._animation = requestAnimationFrame(this.update);
  };
}
```

## Step 3: Create ASCII Animation Component

### 3.1 Component Structure

```typescript
interface ASCIIAnimationProps {
  frames?: string[];
  className?: string;
  fps?: number;
  colorOverlay?: boolean;
}

export default function ASCIIAnimation({
  frames: providedFrames,
  className = "",
  fps = 24,
  colorOverlay = false,
}: ASCIIAnimationProps) {
  const [frames, setFrames] = useState<string[]>([]);
  const [isLoading, setIsLoading] = useState(true);
  const [currentFrame, setCurrentFrame] = useState(0);
  const framesRef = useRef<string[]>([]);
```

### 3.2 Frame Loading Logic

```typescript
useEffect(() => {
  const loadFrames = async () => {
    if (providedFrames) {
      setFrames(providedFrames);
      framesRef.current = providedFrames;
      setIsLoading(false);
      return;
    }

    try {
      const frameFiles = Array.from(
        { length: 60 },
        (_, i) => `frame_${String(i + 1).padStart(4, "0")}.txt`,
      );

      const framePromises = frameFiles.map(async (filename) => {
        const response = await fetch(`/frames/${filename}`);
        if (!response.ok) {
          throw new Error(`Failed to fetch ${filename}: ${response.status}`);
        }
        return await response.text();
      });

      const loadedFrames = await Promise.all(framePromises);
      setFrames(loadedFrames);
      framesRef.current = loadedFrames;
      setCurrentFrame(0);
    } catch (error) {
      console.error("Failed to load ASCII frames:", error);
    } finally {
      setIsLoading(false);
    }
  };

  loadFrames();
}, [providedFrames]);
```

### 3.3 Animation Management

```typescript
const [animationManager] = useState(
  () =>
    new AnimationManager(() => {
      setCurrentFrame((current) => {
        if (framesRef.current.length === 0) return current;
        return (current + 1) % framesRef.current.length;
      });
    }, fps),
);

useEffect(() => {
  if (frames.length === 0) return;

  const reducedMotion =
    window.matchMedia(`(prefers-reduced-motion: reduce)`).matches === true;

  if (reducedMotion) {
    return;
  }

  const handleFocus = () => animationManager.start();
  const handleBlur = () => animationManager.pause();

  window.addEventListener("focus", handleFocus);
  window.addEventListener("blur", handleBlur);

  if (document.visibilityState === "visible") {
    animationManager.start();
  }

  return () => {
    window.removeEventListener("focus", handleFocus);
    window.removeEventListener("blur", handleBlur);
    animationManager.pause();
  };
}, [animationManager, frames.length]);
```

### 3.4 Render with Color Overlay

```typescript
return (
  <div className={`relative font-mono whitespace-pre overflow-hidden text-xs leading-none ${className}`}>
    <div>Frame: {currentFrame + 1}/{frames.length}</div>
    <div className="relative">
      {frames[currentFrame]}

      {/* Color overlay */}
      {colorOverlay && (
        <div
          className="absolute inset-0 pointer-events-none"
          style={{
            background:
              "radial-gradient(circle at center, rgba(143,145,3,1) 0%, rgba(64,64,64,1) 85%)",
            mixBlendMode: "color-dodge",
          }}
        />
      )}
    </div>
  </div>
);
```

## Step 4: Integrate into Page

### 4.1 Import and Use Component

```typescript
import ASCIIAnimation from "@/components/ascii-animation";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";

export default function Home() {
  return (
    <main className="flex items-center flex-col m-auto max-w-6xl px-4">
      <Card className="mb-8">
        <CardHeader>
          <CardTitle>ASCII Animation</CardTitle>
        </CardHeader>
        <CardContent>
          <ASCIIAnimation fps={30} colorOverlay={true} />
        </CardContent>
      </Card>
    </main>
  );
}
```

## Key Performance Optimizations

### 1. **RequestAnimationFrame Timing**
- Uses precise frame timing calculation instead of `setInterval`
- Syncs with browser's refresh rate for smooth animation

### 2. **Efficient State Management**
- Uses `useRef` to avoid closure issues with frame data
- Single state variable for current frame index

### 3. **Smart Loading**
- Loads all frames upfront, no runtime I/O
- Fallback mechanism for missing frames

### 4. **Performance Features**
- Respects `prefers-reduced-motion` 
- Pauses animation when window loses focus
- Uses `pointer-events-none` for overlays

### 5. **CSS Blend Modes**
- `color-dodge` blend mode for vibrant color effects
- Radial gradients positioned at content center
- Minimal DOM manipulation

## Customization Options

- **FPS**: Adjust `fps` prop (24, 30, 60)
- **Colors**: Modify radial gradient colors
- **Blend Modes**: Try `color`, `overlay`, `screen`, `multiply`
- **Threshold**: Adjust `LUMINANCE_THRESHOLD` in `ascii.sh` to remove more/less background
- **Size**: Modify `OUTPUT_COLUMNS` for different resolutions

## File Structure

```
project/
├── ascii.sh                          # Video conversion script
├── public/frames/                     # ASCII frame files
│   ├── frame_0001.txt
│   ├── frame_0002.txt
│   └── ...
├── components/ascii-animation.tsx     # Main component
└── app/page.tsx                      # Integration
```

This approach delivers the same buttery-smooth performance as Ghostty's terminal animations while being highly customizable for different visual effects.

