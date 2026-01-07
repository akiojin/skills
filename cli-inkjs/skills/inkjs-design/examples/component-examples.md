# Component Examples

Practical examples for building Ink.js components.

## Basic Screen Component

```typescript
import React, { useState } from "react";
import { Box, Text, useInput, useApp } from "ink";

interface MainScreenProps {
  onNavigate: (screen: string) => void;
}

export const MainScreen: React.FC<MainScreenProps> = ({ onNavigate }) => {
  const { exit } = useApp();
  const [selectedIndex, setSelectedIndex] = useState(0);

  const menuItems = [
    { label: "View Branches", value: "branches" },
    { label: "Settings", value: "settings" },
    { label: "Exit", value: "exit" },
  ];

  useInput((input, key) => {
    if (key.upArrow) {
      setSelectedIndex(prev => Math.max(0, prev - 1));
    }
    if (key.downArrow) {
      setSelectedIndex(prev => Math.min(menuItems.length - 1, prev + 1));
    }
    if (key.return) {
      const selected = menuItems[selectedIndex];
      if (selected.value === "exit") {
        exit();
      } else {
        onNavigate(selected.value);
      }
    }
    if (key.ctrl && input === "c") {
      exit();
    }
  });

  return (
    <Box flexDirection="column" padding={1}>
      <Box marginBottom={1}>
        <Text bold color="cyan">Main Menu</Text>
      </Box>

      {menuItems.map((item, index) => (
        <Box key={item.value}>
          <Text
            color={index === selectedIndex ? "green" : undefined}
            inverse={index === selectedIndex}
          >
            {index === selectedIndex ? "❯ " : "  "}
            {item.label}
          </Text>
        </Box>
      ))}

      <Box marginTop={1} borderStyle="single" borderColor="gray" padding={1}>
        <Text dimColor>↑↓ Navigate • Enter Select • Ctrl+C Exit</Text>
      </Box>
    </Box>
  );
};
```

## Multi-Screen App

```typescript
import React from "react";
import { Box, render } from "ink";
import { MainScreen } from "./screens/MainScreen";
import { BranchScreen } from "./screens/BranchScreen";
import { SettingsScreen } from "./screens/SettingsScreen";

type ScreenType = "main" | "branches" | "settings";

const App: React.FC = () => {
  const [screenStack, setScreenStack] = React.useState<ScreenType[]>(["main"]);

  const currentScreen = screenStack[screenStack.length - 1];

  const navigateTo = (screen: ScreenType) => {
    setScreenStack(prev => [...prev, screen]);
  };

  const goBack = () => {
    if (screenStack.length > 1) {
      setScreenStack(prev => prev.slice(0, -1));
    }
  };

  const renderScreen = () => {
    switch (currentScreen) {
      case "main":
        return <MainScreen onNavigate={navigateTo} />;
      case "branches":
        return <BranchScreen onBack={goBack} />;
      case "settings":
        return <SettingsScreen onBack={goBack} />;
      default:
        return null;
    }
  };

  return <Box flexDirection="column">{renderScreen()}</Box>;
};

render(<App />);
```

## Interactive List with Selection

```typescript
import React, { useState, useMemo } from "react";
import { Box, Text, useInput, useStdout } from "ink";

interface Branch {
  name: string;
  current: boolean;
  upstream?: string;
}

interface BranchListProps {
  branches: Branch[];
  onSelect: (branch: Branch) => void;
  onBack: () => void;
}

export const BranchList: React.FC<BranchListProps> = ({
  branches,
  onSelect,
  onBack,
}) => {
  const { stdout } = useStdout();
  const [selectedIndex, setSelectedIndex] = useState(0);
  const [filterText, setFilterText] = useState("");
  const [filterMode, setFilterMode] = useState(false);

  // Calculate visible items based on terminal height
  const terminalRows = stdout?.rows ?? 24;
  const HEADER_LINES = 3;
  const FOOTER_LINES = 2;
  const visibleCount = Math.max(5, terminalRows - HEADER_LINES - FOOTER_LINES);

  // Filter branches
  const filteredBranches = useMemo(() => {
    if (!filterText) return branches;
    return branches.filter(b =>
      b.name.toLowerCase().includes(filterText.toLowerCase())
    );
  }, [branches, filterText]);

  // Calculate scroll offset
  const scrollOffset = useMemo(() => {
    const halfVisible = Math.floor(visibleCount / 2);
    let offset = Math.max(0, selectedIndex - halfVisible);
    const maxOffset = Math.max(0, filteredBranches.length - visibleCount);
    return Math.min(offset, maxOffset);
  }, [selectedIndex, visibleCount, filteredBranches.length]);

  const visibleBranches = filteredBranches.slice(
    scrollOffset,
    scrollOffset + visibleCount
  );

  useInput((input, key) => {
    if (filterMode) {
      if (key.escape) {
        setFilterMode(false);
        setFilterText("");
        return;
      }
      if (key.return) {
        setFilterMode(false);
        setSelectedIndex(0);
        return;
      }
      if (key.backspace) {
        setFilterText(prev => prev.slice(0, -1));
        return;
      }
      if (input && input.length === 1 && !key.ctrl) {
        setFilterText(prev => prev + input);
      }
      return;
    }

    if (key.escape) {
      onBack();
      return;
    }
    if (input === "/") {
      setFilterMode(true);
      return;
    }
    if (key.upArrow) {
      setSelectedIndex(prev => Math.max(0, prev - 1));
    }
    if (key.downArrow) {
      setSelectedIndex(prev =>
        Math.min(filteredBranches.length - 1, prev + 1)
      );
    }
    if (key.return && filteredBranches[selectedIndex]) {
      onSelect(filteredBranches[selectedIndex]);
    }
  });

  return (
    <Box flexDirection="column" padding={1}>
      {/* Header */}
      <Box marginBottom={1}>
        <Text bold color="cyan">Branches</Text>
        <Text dimColor> ({filteredBranches.length})</Text>
      </Box>

      {/* Filter input */}
      {filterMode && (
        <Box marginBottom={1}>
          <Text>Filter: </Text>
          <Text color="yellow">{filterText}</Text>
          <Text>_</Text>
        </Box>
      )}

      {/* Branch list */}
      <Box flexDirection="column" height={visibleCount}>
        {visibleBranches.map((branch, i) => {
          const actualIndex = scrollOffset + i;
          const isSelected = actualIndex === selectedIndex;

          return (
            <Box key={branch.name}>
              <Text
                color={branch.current ? "green" : undefined}
                inverse={isSelected}
              >
                {isSelected ? "❯ " : "  "}
                {branch.current ? "* " : "  "}
                {branch.name}
              </Text>
              {branch.upstream && (
                <Text dimColor> → {branch.upstream}</Text>
              )}
            </Box>
          );
        })}
      </Box>

      {/* Footer */}
      <Box marginTop={1} borderStyle="single" borderColor="gray" padding={1}>
        <Text dimColor>
          ↑↓ Navigate • Enter Select • / Filter • Esc Back
        </Text>
      </Box>
    </Box>
  );
};
```

## Progress Indicator

```typescript
import React, { useState, useEffect, useRef } from "react";
import { Box, Text } from "ink";

interface ProgressIndicatorProps {
  tasks: string[];
  onComplete: () => void;
}

const SPINNER_FRAMES = ["⠋", "⠙", "⠹", "⠸", "⠼", "⠴", "⠦", "⠧", "⠇", "⠏"];

export const ProgressIndicator: React.FC<ProgressIndicatorProps> = ({
  tasks,
  onComplete,
}) => {
  const [currentTask, setCurrentTask] = useState(0);
  const [completedTasks, setCompletedTasks] = useState<Set<number>>(new Set());
  const [spinnerFrame, setSpinnerFrame] = useState(0);
  const startTime = useRef(Date.now());

  // Spinner animation
  useEffect(() => {
    const timer = setInterval(() => {
      setSpinnerFrame(prev => (prev + 1) % SPINNER_FRAMES.length);
    }, 80);
    return () => clearInterval(timer);
  }, []);

  // Simulate task processing
  useEffect(() => {
    if (currentTask >= tasks.length) {
      onComplete();
      return;
    }

    const timer = setTimeout(() => {
      setCompletedTasks(prev => new Set([...prev, currentTask]));
      setCurrentTask(prev => prev + 1);
    }, 1000 + Math.random() * 2000);

    return () => clearTimeout(timer);
  }, [currentTask, tasks.length, onComplete]);

  const progress = (completedTasks.size / tasks.length) * 100;
  const elapsed = Math.floor((Date.now() - startTime.current) / 1000);

  return (
    <Box flexDirection="column" padding={1}>
      <Box marginBottom={1}>
        <Text bold>Processing Tasks</Text>
      </Box>

      {/* Progress bar */}
      <Box marginBottom={1}>
        <Text color="green">
          {"█".repeat(Math.floor(progress / 5))}
        </Text>
        <Text color="gray">
          {"░".repeat(20 - Math.floor(progress / 5))}
        </Text>
        <Text> {progress.toFixed(0)}%</Text>
      </Box>

      {/* Task list */}
      <Box flexDirection="column" marginBottom={1}>
        {tasks.map((task, i) => {
          const isCompleted = completedTasks.has(i);
          const isCurrent = i === currentTask;

          return (
            <Box key={i}>
              {isCompleted && <Text color="green">✓ </Text>}
              {isCurrent && (
                <Text color="cyan">{SPINNER_FRAMES[spinnerFrame]} </Text>
              )}
              {!isCompleted && !isCurrent && <Text dimColor>○ </Text>}
              <Text dimColor={!isCompleted && !isCurrent}>{task}</Text>
            </Box>
          );
        })}
      </Box>

      {/* Status */}
      <Box>
        <Text dimColor>
          Elapsed: {elapsed}s • {completedTasks.size}/{tasks.length} completed
        </Text>
      </Box>
    </Box>
  );
};
```

## Reusable Select Component

```typescript
import React, { useState, useMemo } from "react";
import { Box, Text, useInput } from "ink";

interface SelectItem<T> {
  label: string;
  value: T;
  disabled?: boolean;
}

interface SelectProps<T> {
  items: SelectItem<T>[];
  value?: T;
  onChange?: (value: T) => void;
  defaultValue?: T;
  onSelect?: (value: T) => void;
  disabled?: boolean;
}

function arePropsEqual<T>(
  prev: SelectProps<T>,
  next: SelectProps<T>
): boolean {
  if (prev.items.length !== next.items.length) return false;
  if (prev.value !== next.value) return false;
  if (prev.disabled !== next.disabled) return false;
  for (let i = 0; i < prev.items.length; i++) {
    if (prev.items[i].value !== next.items[i].value) return false;
    if (prev.items[i].label !== next.items[i].label) return false;
  }
  return true;
}

function SelectComponent<T>({
  items,
  value,
  onChange,
  defaultValue,
  onSelect,
  disabled = false,
}: SelectProps<T>) {
  const isControlled = value !== undefined;
  const [internalIndex, setInternalIndex] = useState(() => {
    const initial = isControlled ? value : defaultValue;
    return items.findIndex(item => item.value === initial);
  });

  const selectedIndex = useMemo(() => {
    if (isControlled) {
      return items.findIndex(item => item.value === value);
    }
    return internalIndex;
  }, [isControlled, items, value, internalIndex]);

  useInput((input, key) => {
    if (disabled) return;

    if (key.upArrow) {
      const newIndex = Math.max(0, selectedIndex - 1);
      if (!isControlled) setInternalIndex(newIndex);
      onChange?.(items[newIndex].value);
    }

    if (key.downArrow) {
      const newIndex = Math.min(items.length - 1, selectedIndex + 1);
      if (!isControlled) setInternalIndex(newIndex);
      onChange?.(items[newIndex].value);
    }

    if (key.return) {
      const item = items[selectedIndex];
      if (!item.disabled) {
        onSelect?.(item.value);
      }
    }
  });

  return (
    <Box flexDirection="column">
      {items.map((item, index) => (
        <Box key={String(item.value)}>
          <Text
            color={item.disabled ? "gray" : index === selectedIndex ? "green" : undefined}
            inverse={index === selectedIndex}
            dimColor={item.disabled}
          >
            {index === selectedIndex ? "❯ " : "  "}
            {item.label}
          </Text>
        </Box>
      ))}
    </Box>
  );
}

export const Select = React.memo(SelectComponent, arePropsEqual) as typeof SelectComponent;
```

## Usage Tips

1. **Start with Screen components** - Build top-level screens first
2. **Extract reusable parts** - Move common UI to Part components
3. **Use TypeScript** - Strong typing prevents runtime errors
4. **Test with ink-testing-library** - Verify keyboard interactions
5. **Handle edge cases** - Empty lists, long text, small terminals
