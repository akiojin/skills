# Animation Patterns

Patterns for implementing animations in Ink.js applications.

## Overview

CLI animations enhance user experience during loading states and long-running operations. This guide covers spinners, progress bars, and frame-based animations.

## Spinner Animation

### Basic Spinner Frames

```typescript
const SPINNER_FRAMES = ["⠋", "⠙", "⠹", "⠸", "⠼", "⠴", "⠦", "⠧", "⠇", "⠏"];
const SPINNER_INTERVAL = 80; // milliseconds
```

### SpinnerIcon Component

```typescript
import React, { useState, useEffect } from "react";
import { Text } from "ink";

interface SpinnerIconProps {
  color?: string;
}

export const SpinnerIcon: React.FC<SpinnerIconProps> = ({ color = "cyan" }) => {
  const [frameIndex, setFrameIndex] = useState(0);

  useEffect(() => {
    const timer = setInterval(() => {
      setFrameIndex(prev => (prev + 1) % SPINNER_FRAMES.length);
    }, SPINNER_INTERVAL);

    return () => clearInterval(timer);
  }, []);

  return <Text color={color}>{SPINNER_FRAMES[frameIndex]}</Text>;
};
```

### useSpinnerFrame Hook

Extract animation logic to a reusable hook:

```typescript
import { useState, useEffect } from "react";

export function useSpinnerFrame(
  frames: string[] = SPINNER_FRAMES,
  interval: number = SPINNER_INTERVAL
): string {
  const [frameIndex, setFrameIndex] = useState(0);

  useEffect(() => {
    const timer = setInterval(() => {
      setFrameIndex(prev => (prev + 1) % frames.length);
    }, interval);

    return () => clearInterval(timer);
  }, [frames.length, interval]);

  return frames[frameIndex];
}
```

Usage:

```typescript
const LoadingIndicator: React.FC = () => {
  const spinnerFrame = useSpinnerFrame();
  return <Text>{spinnerFrame} Loading...</Text>;
};
```

## Progress Bar

### Basic Progress Bar

```typescript
import React from "react";
import { Box, Text } from "ink";

interface ProgressBarProps {
  percent: number;  // 0-100
  width?: number;
  showPercent?: boolean;
}

export const ProgressBar: React.FC<ProgressBarProps> = ({
  percent,
  width = 40,
  showPercent = true,
}) => {
  const filled = Math.round((percent / 100) * width);
  const empty = width - filled;

  return (
    <Box>
      <Text color="green">{"█".repeat(filled)}</Text>
      <Text color="gray">{"░".repeat(empty)}</Text>
      {showPercent && (
        <Text> {percent.toFixed(0)}%</Text>
      )}
    </Box>
  );
};
```

### Progress with Phases

```typescript
interface PhaseProgress {
  phase: string;
  current: number;
  total: number;
}

export const PhaseProgressBar: React.FC<{ progress: PhaseProgress }> = ({
  progress,
}) => {
  const percent = (progress.current / progress.total) * 100;

  return (
    <Box flexDirection="column">
      <Text bold>{progress.phase}</Text>
      <Box>
        <ProgressBar percent={percent} />
        <Text dimColor> ({progress.current}/{progress.total})</Text>
      </Box>
    </Box>
  );
};
```

### Progress with Remaining Time

```typescript
import { useState, useEffect, useRef } from "react";

interface ProgressWithTimeProps {
  current: number;
  total: number;
}

export const ProgressWithTime: React.FC<ProgressWithTimeProps> = ({
  current,
  total,
}) => {
  const startTime = useRef(Date.now());
  const [remainingTime, setRemainingTime] = useState<string>("");

  useEffect(() => {
    if (current === 0) {
      startTime.current = Date.now();
      return;
    }

    const elapsed = Date.now() - startTime.current;
    const rate = current / elapsed;
    const remaining = (total - current) / rate;

    if (remaining > 0 && isFinite(remaining)) {
      const seconds = Math.ceil(remaining / 1000);
      if (seconds < 60) {
        setRemainingTime(`${seconds}s`);
      } else {
        const minutes = Math.floor(seconds / 60);
        const secs = seconds % 60;
        setRemainingTime(`${minutes}m ${secs}s`);
      }
    }
  }, [current, total]);

  const percent = (current / total) * 100;

  return (
    <Box>
      <ProgressBar percent={percent} />
      {remainingTime && <Text dimColor> ~{remainingTime} remaining</Text>}
    </Box>
  );
};
```

## Frame-Based Status Animations

### Status-Dependent Animation

```typescript
type Status = "idle" | "loading" | "success" | "error";

const STATUS_FRAMES: Record<Status, string[]> = {
  idle: ["○"],
  loading: ["⠋", "⠙", "⠹", "⠸", "⠼", "⠴", "⠦", "⠧", "⠇", "⠏"],
  success: ["✓"],
  error: ["✗"],
};

const STATUS_COLORS: Record<Status, string> = {
  idle: "gray",
  loading: "cyan",
  success: "green",
  error: "red",
};

export const StatusIcon: React.FC<{ status: Status }> = ({ status }) => {
  const frames = STATUS_FRAMES[status];
  const frame = useSpinnerFrame(frames, status === "loading" ? 80 : 1000);

  return <Text color={STATUS_COLORS[status]}>{frame}</Text>;
};
```

### Animated Text Effects

```typescript
const TYPING_INTERVAL = 50;

export const TypedText: React.FC<{ text: string }> = ({ text }) => {
  const [visibleLength, setVisibleLength] = useState(0);

  useEffect(() => {
    if (visibleLength < text.length) {
      const timer = setTimeout(() => {
        setVisibleLength(prev => prev + 1);
      }, TYPING_INTERVAL);
      return () => clearTimeout(timer);
    }
  }, [visibleLength, text.length]);

  return <Text>{text.slice(0, visibleLength)}</Text>;
};
```

## Performance Considerations

### Isolate Spinner Re-renders

Prevent parent component re-renders by isolating animation state:

```typescript
// Bad: Parent re-renders on every frame
const Parent: React.FC = () => {
  const frame = useSpinnerFrame();
  return (
    <Box>
      <Text>{frame}</Text>
      <ExpensiveComponent /> {/* Re-renders unnecessarily */}
    </Box>
  );
};

// Good: Spinner is isolated
const Parent: React.FC = () => {
  return (
    <Box>
      <SpinnerIcon /> {/* Animation isolated */}
      <ExpensiveComponent /> {/* Stable */}
    </Box>
  );
};
```

### Conditional Animation

Stop animations when not visible:

```typescript
export const ConditionalSpinner: React.FC<{ active: boolean }> = ({ active }) => {
  const [frameIndex, setFrameIndex] = useState(0);

  useEffect(() => {
    if (!active) return;

    const timer = setInterval(() => {
      setFrameIndex(prev => (prev + 1) % SPINNER_FRAMES.length);
    }, SPINNER_INTERVAL);

    return () => clearInterval(timer);
  }, [active]);

  if (!active) return null;

  return <Text color="cyan">{SPINNER_FRAMES[frameIndex]}</Text>;
};
```

## Using @inkjs/ui Components

The `@inkjs/ui` package provides pre-built animated components:

```typescript
import { Spinner, ProgressBar } from "@inkjs/ui";

// Spinner with label
<Spinner label="Loading..." />

// Progress bar (0-100)
<ProgressBar value={75} />
```

## Best Practices

1. **Use appropriate frame rates** - 80-100ms for spinners, slower for progress
2. **Isolate animation state** - Prevent unnecessary parent re-renders
3. **Stop animations when hidden** - Save CPU cycles
4. **Provide text context** - Always include labels with spinners
5. **Show progress when possible** - Users prefer progress over indefinite spinners
6. **Handle completion state** - Show success/error indicators when done
