# Multi-Screen Navigation

Patterns for managing multiple screens in Ink.js applications.

## Overview

Multi-screen navigation in CLI applications follows a stack-based model, similar to mobile app navigation. This allows users to drill down into detail views and return to previous screens.

## ScreenType Definition

Define all possible screens as a union type:

```typescript
export type ScreenType =
  | "main-menu"
  | "branch-list"
  | "branch-creator"
  | "settings"
  | "help";
```

## useScreenState Hook

Create a reusable hook for screen stack management:

```typescript
import { useState, useCallback } from "react";

export function useScreenState<T extends string>(initialScreen: T) {
  const [screenStack, setScreenStack] = useState<T[]>([initialScreen]);

  const currentScreen = screenStack[screenStack.length - 1];

  const navigateTo = useCallback((screen: T) => {
    setScreenStack(prev => [...prev, screen]);
  }, []);

  const goBack = useCallback(() => {
    if (screenStack.length > 1) {
      setScreenStack(prev => prev.slice(0, -1));
    }
  }, [screenStack.length]);

  const reset = useCallback((screen?: T) => {
    setScreenStack([screen ?? initialScreen]);
  }, [initialScreen]);

  const canGoBack = screenStack.length > 1;

  return {
    currentScreen,
    screenStack,
    navigateTo,
    goBack,
    reset,
    canGoBack,
  };
}
```

## App.tsx Pattern

Use switch-case for screen rendering:

```typescript
import React from "react";
import { Box } from "ink";
import { useScreenState } from "../hooks/useScreenState";
import { MainMenuScreen } from "./screens/MainMenuScreen";
import { BranchListScreen } from "./screens/BranchListScreen";
import { SettingsScreen } from "./screens/SettingsScreen";

type ScreenType = "main-menu" | "branch-list" | "settings";

export const App: React.FC = () => {
  const {
    currentScreen,
    navigateTo,
    goBack,
    canGoBack,
  } = useScreenState<ScreenType>("main-menu");

  const renderScreen = () => {
    switch (currentScreen) {
      case "main-menu":
        return (
          <MainMenuScreen
            onSelect={(item) => navigateTo(item as ScreenType)}
          />
        );

      case "branch-list":
        return (
          <BranchListScreen
            onBack={goBack}
            onSelect={(branch) => {
              // Handle branch selection
            }}
          />
        );

      case "settings":
        return (
          <SettingsScreen
            onBack={goBack}
            onSave={(settings) => {
              // Save settings and go back
              goBack();
            }}
          />
        );

      default:
        return null;
    }
  };

  return (
    <Box flexDirection="column">
      {renderScreen()}
    </Box>
  );
};
```

## Data Passing Between Screens

For passing data between screens, extend the screen state:

```typescript
interface ScreenState {
  screen: ScreenType;
  data?: Record<string, unknown>;
}

export function useScreenStateWithData<T extends string>(initialScreen: T) {
  const [stack, setStack] = useState<ScreenState[]>([
    { screen: initialScreen }
  ]);

  const current = stack[stack.length - 1];

  const navigateTo = useCallback((screen: T, data?: Record<string, unknown>) => {
    setStack(prev => [...prev, { screen, data }]);
  }, []);

  const goBack = useCallback((returnData?: Record<string, unknown>) => {
    if (stack.length > 1) {
      setStack(prev => {
        const newStack = prev.slice(0, -1);
        if (returnData) {
          const last = newStack[newStack.length - 1];
          newStack[newStack.length - 1] = {
            ...last,
            data: { ...last.data, ...returnData }
          };
        }
        return newStack;
      });
    }
  }, [stack.length]);

  return {
    currentScreen: current.screen as T,
    screenData: current.data,
    navigateTo,
    goBack,
  };
}
```

## Keyboard Navigation

Implement global back navigation:

```typescript
import { useInput, useApp } from "ink";

export const ScreenContainer: React.FC<{
  onBack?: () => void;
  canGoBack: boolean;
  children: React.ReactNode;
}> = ({ onBack, canGoBack, children }) => {
  const { exit } = useApp();

  useInput((input, key) => {
    // Escape to go back
    if (key.escape && canGoBack && onBack) {
      onBack();
      return;
    }

    // Ctrl+C to exit
    if (key.ctrl && input === "c") {
      exit();
    }
  });

  return <>{children}</>;
};
```

## Tab-Based Navigation

For tab-style navigation between screens:

```typescript
const SCREENS: ScreenType[] = ["main", "list", "settings"];

const [currentIndex, setCurrentIndex] = useState(0);

useInput((input, key) => {
  if (key.tab) {
    if (key.shift) {
      // Shift+Tab: previous screen
      setCurrentIndex(prev =>
        prev === 0 ? SCREENS.length - 1 : prev - 1
      );
    } else {
      // Tab: next screen
      setCurrentIndex(prev =>
        prev === SCREENS.length - 1 ? 0 : prev + 1
      );
    }
  }
});
```

## Screen Transition with Confirmation

Handle protected navigation (e.g., unsaved changes):

```typescript
const [hasUnsavedChanges, setHasUnsavedChanges] = useState(false);
const [showConfirmDialog, setShowConfirmDialog] = useState(false);
const [pendingNavigation, setPendingNavigation] = useState<ScreenType | null>(null);

const safeNavigate = (screen: ScreenType) => {
  if (hasUnsavedChanges) {
    setPendingNavigation(screen);
    setShowConfirmDialog(true);
  } else {
    navigateTo(screen);
  }
};

const handleConfirm = (confirmed: boolean) => {
  if (confirmed && pendingNavigation) {
    setHasUnsavedChanges(false);
    navigateTo(pendingNavigation);
  }
  setPendingNavigation(null);
  setShowConfirmDialog(false);
};
```

## Best Practices

1. **Keep screen stack shallow** - Avoid deep nesting; use modals for overlays
2. **Reset state on navigation** - Clear form data when leaving screens
3. **Handle edge cases** - Prevent going back from root screen
4. **Persist important state** - Save user preferences across screen changes
5. **Use consistent shortcuts** - Escape for back, Tab for switching
6. **Show navigation hints** - Display available shortcuts in footer
