---
title: wms
date: 2020-03-16 15:16:23
tags: ["源码分析"]
cover: https://images.unsplash.com/photo-1613470790260-582059be2d4d?ixid=MXwxMjA3fDB8MHx0b3BpYy1mZWVkfDZ8Ym84alFLVGFFMFl8fGVufDB8fHw%3D&ixlib=rb-1.2.1&auto=format&fit=crop&w=500&q=60
---



## Window

## WindowManagerService

```java
//frameworks/base/services/core/java/com/android/server/wm/WindowManagerService.java
public static WindowManagerService main(final Context context, final InputManagerService im,
            final boolean haveInputMethods, final boolean showBootMsgs, final boolean onlyCore,
            WindowManagerPolicy policy) {
    DisplayThread.getHandler().runWithScissors(() ->
            sInstance = new WindowManagerService(context, im, haveInputMethods, showBootMsgs,
                    onlyCore, policy), 0);
    return sInstance;
}
```

```java
//frameworks/base/services/core/java/com/android/server/wm/WindowManagerService.java
public IWindowSession openSession(IWindowSessionCallback callback, IInputMethodClient client,
            IInputContext inputContext) {
      if (client == null) throw new IllegalArgumentException("null client");
      if (inputContext == null) throw new IllegalArgumentException("null inputContext");
      Session session = new Session(this, callback, client, inputContext); //创建session并传入this
      return session;
}
```



```java
public int addWindow(Session session, IWindow client, int seq,
            WindowManager.LayoutParams attrs, int viewVisibility, int displayId,
            Rect outContentInsets, Rect outStableInsets, Rect outOutsets,
            InputChannel outInputChannel) {
        int[] appOp = new int[1];
        int res = mPolicy.checkAddPermission(attrs, appOp);
        if (res != WindowManagerGlobal.ADD_OKAY) {
            return res;
        }

        boolean reportNewConfig = false;
        WindowState attachedWindow = null;
        long origId;
        final int callingUid = Binder.getCallingUid();
        final int type = attrs.type;

        synchronized(mWindowMap) {
            if (!mDisplayReady) {
                throw new IllegalStateException("Display has not been initialialized");
            }

            final DisplayContent displayContent = getDisplayContentLocked(displayId);
            if (displayContent == null) {
                Slog.w(TAG_WM, "Attempted to add window to a display that does not exist: "
                        + displayId + ".  Aborting.");
                return WindowManagerGlobal.ADD_INVALID_DISPLAY;
            }
            if (!displayContent.hasAccess(session.mUid)) {
                Slog.w(TAG_WM, "Attempted to add window to a display for which the application "
                        + "does not have access: " + displayId + ".  Aborting.");
                return WindowManagerGlobal.ADD_INVALID_DISPLAY;
            }

            if (mWindowMap.containsKey(client.asBinder())) {
                Slog.w(TAG_WM, "Window " + client + " is already added");
                return WindowManagerGlobal.ADD_DUPLICATE_ADD;
            }

            if (type >= FIRST_SUB_WINDOW && type <= LAST_SUB_WINDOW) {
                attachedWindow = windowForClientLocked(null, attrs.token, false);
                if (attachedWindow == null) {
                    Slog.w(TAG_WM, "Attempted to add window with token that is not a window: "
                          + attrs.token + ".  Aborting.");
                    return WindowManagerGlobal.ADD_BAD_SUBWINDOW_TOKEN;
                }
                if (attachedWindow.mAttrs.type >= FIRST_SUB_WINDOW
                        && attachedWindow.mAttrs.type <= LAST_SUB_WINDOW) {
                    Slog.w(TAG_WM, "Attempted to add window with token that is a sub-window: "
                            + attrs.token + ".  Aborting.");
                    return WindowManagerGlobal.ADD_BAD_SUBWINDOW_TOKEN;
                }
            }

            if (type == TYPE_PRIVATE_PRESENTATION && !displayContent.isPrivate()) {
                Slog.w(TAG_WM, "Attempted to add private presentation window to a non-private display.  Aborting.");
                return WindowManagerGlobal.ADD_PERMISSION_DENIED;
            }

            boolean addToken = false;
            WindowToken token = mTokenMap.get(attrs.token);
            AppWindowToken atoken = null;
            boolean addToastWindowRequiresToken = false;

            if (token == null) {
                if (type >= FIRST_APPLICATION_WINDOW && type <= LAST_APPLICATION_WINDOW) {
                    Slog.w(TAG_WM, "Attempted to add application window with unknown token "
                          + attrs.token + ".  Aborting.");
                    return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
                if (type == TYPE_INPUT_METHOD) {
                    Slog.w(TAG_WM, "Attempted to add input method window with unknown token "
                          + attrs.token + ".  Aborting.");
                    return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
                if (type == TYPE_VOICE_INTERACTION) {
                    Slog.w(TAG_WM, "Attempted to add voice interaction window with unknown token "
                          + attrs.token + ".  Aborting.");
                    return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
                if (type == TYPE_WALLPAPER) {
                    Slog.w(TAG_WM, "Attempted to add wallpaper window with unknown token "
                          + attrs.token + ".  Aborting.");
                    return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
                if (type == TYPE_DREAM) {
                    Slog.w(TAG_WM, "Attempted to add Dream window with unknown token "
                          + attrs.token + ".  Aborting.");
                    return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
                if (type == TYPE_QS_DIALOG) {
                    Slog.w(TAG_WM, "Attempted to add QS dialog window with unknown token "
                          + attrs.token + ".  Aborting.");
                    return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
                if (type == TYPE_ACCESSIBILITY_OVERLAY) {
                    Slog.w(TAG_WM, "Attempted to add Accessibility overlay window with unknown token "
                            + attrs.token + ".  Aborting.");
                    return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
                if (type == TYPE_TOAST) {
                    // Apps targeting SDK above N MR1 cannot arbitrary add toast windows.
                    if (doesAddToastWindowRequireToken(attrs.packageName, callingUid,
                            attachedWindow)) {
                        Slog.w(TAG_WM, "Attempted to add a toast window with unknown token "
                                + attrs.token + ".  Aborting.");
                        return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                    }
                }
                token = new WindowToken(this, attrs.token, -1, false);
                addToken = true;
            } else if (type >= FIRST_APPLICATION_WINDOW && type <= LAST_APPLICATION_WINDOW) {
                atoken = token.appWindowToken;
                if (atoken == null) {
                    Slog.w(TAG_WM, "Attempted to add window with non-application token "
                          + token + ".  Aborting.");
                    return WindowManagerGlobal.ADD_NOT_APP_TOKEN;
                } else if (atoken.removed) {
                    Slog.w(TAG_WM, "Attempted to add window with exiting application token "
                          + token + ".  Aborting.");
                    return WindowManagerGlobal.ADD_APP_EXITING;
                }
                if (type == TYPE_APPLICATION_STARTING && atoken.firstWindowDrawn) {
                    // No need for this guy!
                    if (DEBUG_STARTING_WINDOW || localLOGV) Slog.v(
                            TAG_WM, "**** NO NEED TO START: " + attrs.getTitle());
                    return WindowManagerGlobal.ADD_STARTING_NOT_NEEDED;
                }
            } else if (type == TYPE_INPUT_METHOD) {
                if (token.windowType != TYPE_INPUT_METHOD) {
                    Slog.w(TAG_WM, "Attempted to add input method window with bad token "
                            + attrs.token + ".  Aborting.");
                      return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
            } else if (type == TYPE_VOICE_INTERACTION) {
                if (token.windowType != TYPE_VOICE_INTERACTION) {
                    Slog.w(TAG_WM, "Attempted to add voice interaction window with bad token "
                            + attrs.token + ".  Aborting.");
                      return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
            } else if (type == TYPE_WALLPAPER) {
                if (token.windowType != TYPE_WALLPAPER) {
                    Slog.w(TAG_WM, "Attempted to add wallpaper window with bad token "
                            + attrs.token + ".  Aborting.");
                      return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
            } else if (type == TYPE_DREAM) {
                if (token.windowType != TYPE_DREAM) {
                    Slog.w(TAG_WM, "Attempted to add Dream window with bad token "
                            + attrs.token + ".  Aborting.");
                      return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
            } else if (type == TYPE_ACCESSIBILITY_OVERLAY) {
                if (token.windowType != TYPE_ACCESSIBILITY_OVERLAY) {
                    Slog.w(TAG_WM, "Attempted to add Accessibility overlay window with bad token "
                            + attrs.token + ".  Aborting.");
                    return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
            } else if (type == TYPE_TOAST) {
                // Apps targeting SDK above N MR1 cannot arbitrary add toast windows.
                addToastWindowRequiresToken = doesAddToastWindowRequireToken(attrs.packageName,
                        callingUid, attachedWindow);
                if (addToastWindowRequiresToken && token.windowType != TYPE_TOAST) {
                    Slog.w(TAG_WM, "Attempted to add a toast window with bad token "
                            + attrs.token + ".  Aborting.");
                    return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
            } else if (type == TYPE_QS_DIALOG) {
                if (token.windowType != TYPE_QS_DIALOG) {
                    Slog.w(TAG_WM, "Attempted to add QS dialog window with bad token "
                            + attrs.token + ".  Aborting.");
                    return WindowManagerGlobal.ADD_BAD_APP_TOKEN;
                }
            } else if (token.appWindowToken != null) {
                Slog.w(TAG_WM, "Non-null appWindowToken for system window of type=" + type);
                // It is not valid to use an app token with other system types; we will
                // instead make a new token for it (as if null had been passed in for the token).
                attrs.token = null;
                token = new WindowToken(this, null, -1, false);
                addToken = true;
            }
           //创建WindowState
            WindowState win = new (this, session, client, token,
                    attachedWindow, appOp[0], seq, attrs, viewVisibility, displayContent);
            if (win.mDeathRecipient == null) {
                // Client has apparently died, so there is no reason to
                // continue.
                Slog.w(TAG_WM, "Adding window client " + client.asBinder()
                        + " that is dead, aborting.");
                return WindowManagerGlobal.ADD_APP_EXITING;
            }

            if (win.getDisplayContent() == null) {
                Slog.w(TAG_WM, "Adding window to Display that has been removed.");
                return WindowManagerGlobal.ADD_INVALID_DISPLAY;
            }
            //调整WindowManager的LayoutParams参数
            mPolicy.adjustWindowParamsLw(win.mAttrs);
            win.setShowToOwnerOnlyLocked(mPolicy.checkShowToOwnerOnly(attrs));

            res = mPolicy.prepareAddWindowLw(win, attrs);
            if (res != WindowManagerGlobal.ADD_OKAY) {
                return res;
            }

            final boolean openInputChannels = (outInputChannel != null
                    && (attrs.inputFeatures & INPUT_FEATURE_NO_INPUT_CHANNEL) == 0);
            if  (openInputChannels) {
                win.openInputChannel(outInputChannel);
            }

            // If adding a toast requires a token for this app we always schedule hiding
            // toast windows to make sure they don't stick around longer then necessary.
            // We hide instead of remove such windows as apps aren't prepared to handle
            // windows being removed under them.
            //
            // If the app is older it can add toasts without a token and hence overlay
            // other apps. To be maximally compatible with these apps we will hide the
            // window after the toast timeout only if the focused window is from another
            // UID, otherwise we allow unlimited duration. When a UID looses focus we
            // schedule hiding all of its toast windows.
            if (type == TYPE_TOAST) {
                if (!getDefaultDisplayContentLocked().canAddToastWindowForUid(callingUid)) {
                    Slog.w(TAG_WM, "Adding more than one toast window for UID at a time.");
                    return WindowManagerGlobal.ADD_DUPLICATE_ADD;
                }
                // Make sure this happens before we moved focus as one can make the
                // toast focusable to force it not being hidden after the timeout.
                // Focusable toasts are always timed out to prevent a focused app to
                // show a focusable toasts while it has focus which will be kept on
                // the screen after the activity goes away.
                if (addToastWindowRequiresToken
                        || (attrs.flags & LayoutParams.FLAG_NOT_FOCUSABLE) == 0
                        || mCurrentFocus == null
                        || mCurrentFocus.mOwnerUid != callingUid) {
                    mH.sendMessageDelayed(
                            mH.obtainMessage(H.WINDOW_HIDE_TIMEOUT, win),
                            win.mAttrs.hideTimeoutMilliseconds);
                }
            }

            // From now on, no exceptions or errors allowed!

            res = WindowManagerGlobal.ADD_OKAY;

            if (excludeWindowTypeFromTapOutTask(type)) {
                displayContent.mTapExcludedWindows.add(win);
            }

            origId = Binder.clearCallingIdentity();

            if (addToken) {
                mTokenMap.put(attrs.token, token);
            }
            win.attach();
            mWindowMap.put(client.asBinder(), win);
            if (win.mAppOp != AppOpsManager.OP_NONE) {
                int startOpResult = mAppOps.startOpNoThrow(win.mAppOp, win.getOwningUid(),
                        win.getOwningPackage());
                if ((startOpResult != AppOpsManager.MODE_ALLOWED) &&
                        (startOpResult != AppOpsManager.MODE_DEFAULT)) {
                    win.setAppOpVisibilityLw(false);
                }
            }

            final boolean hideSystemAlertWindows = !mHidingNonSystemOverlayWindows.isEmpty();
            win.setForceHideNonSystemOverlayWindowIfNeeded(hideSystemAlertWindows);

            if (type == TYPE_APPLICATION_STARTING && token.appWindowToken != null) {
                token.appWindowToken.startingWindow = win;
                if (DEBUG_STARTING_WINDOW) Slog.v (TAG_WM, "addWindow: " + token.appWindowToken
                        + " startingWindow=" + win);
            }

            boolean imMayMove = true;

            if (type == TYPE_INPUT_METHOD) {
                win.mGivenInsetsPending = true;
                mInputMethodWindow = win;
                addInputMethodWindowToListLocked(win);
                imMayMove = false;
            } else if (type == TYPE_INPUT_METHOD_DIALOG) {
                mInputMethodDialogs.add(win);
                addWindowToListInOrderLocked(win, true);
                moveInputMethodDialogsLocked(findDesiredInputMethodWindowIndexLocked(true));
                imMayMove = false;
            } else {
                addWindowToListInOrderLocked(win, true);
                if (type == TYPE_WALLPAPER) {
                    mWallpaperControllerLocked.clearLastWallpaperTimeoutTime();
                    displayContent.pendingLayoutChanges |= FINISH_LAYOUT_REDO_WALLPAPER;
                } else if ((attrs.flags&FLAG_SHOW_WALLPAPER) != 0) {
                    displayContent.pendingLayoutChanges |= FINISH_LAYOUT_REDO_WALLPAPER;
                } else if (mWallpaperControllerLocked.isBelowWallpaperTarget(win)) {
                    // If there is currently a wallpaper being shown, and
                    // the base layer of the new window is below the current
                    // layer of the target window, then adjust the wallpaper.
                    // This is to avoid a new window being placed between the
                    // wallpaper and its target.
                    displayContent.pendingLayoutChanges |= FINISH_LAYOUT_REDO_WALLPAPER;
                }
            }

            // If the window is being added to a task that's docked but non-resizeable,
            // we need to update this new window's scroll position when it's added.
            win.applyScrollIfNeeded();

            // If the window is being added to a stack that's currently adjusted for IME,
            // make sure to apply the same adjust to this new window.
            win.applyAdjustForImeIfNeeded();

            if (type == TYPE_DOCK_DIVIDER) {
                getDefaultDisplayContentLocked().getDockedDividerController().setWindow(win);
            }

            final WindowStateAnimator winAnimator = win.mWinAnimator;
            winAnimator.mEnterAnimationPending = true;
            winAnimator.mEnteringAnimation = true;
            // Check if we need to prepare a transition for replacing window first.
            if (atoken != null && !prepareWindowReplacementTransition(atoken)) {
                // If not, check if need to set up a dummy transition during display freeze
                // so that the unfreeze wait for the apps to draw. This might be needed if
                // the app is relaunching.
                prepareNoneTransitionForRelaunching(atoken);
            }

            if (displayContent.isDefaultDisplay) {
                final DisplayInfo displayInfo = displayContent.getDisplayInfo();
                final Rect taskBounds;
                if (atoken != null && atoken.mTask != null) {
                    taskBounds = mTmpRect;
                    atoken.mTask.getBounds(mTmpRect);
                } else {
                    taskBounds = null;
                }
                if (mPolicy.getInsetHintLw(win.mAttrs, taskBounds, mRotation,
                        displayInfo.logicalWidth, displayInfo.logicalHeight, outContentInsets,
                        outStableInsets, outOutsets)) {
                    res |= WindowManagerGlobal.ADD_FLAG_ALWAYS_CONSUME_NAV_BAR;
                }
            } else {
                outContentInsets.setEmpty();
                outStableInsets.setEmpty();
            }

            if (mInTouchMode) {
                res |= WindowManagerGlobal.ADD_FLAG_IN_TOUCH_MODE;
            }
            if (win.mAppToken == null || !win.mAppToken.clientHidden) {
                res |= WindowManagerGlobal.ADD_FLAG_APP_VISIBLE;
            }

            mInputMonitor.setUpdateInputWindowsNeededLw();

            boolean focusChanged = false;
            if (win.canReceiveKeys()) {
                focusChanged = updateFocusedWindowLocked(UPDATE_FOCUS_WILL_ASSIGN_LAYERS,
                        false /*updateInputWindows*/);
                if (focusChanged) {
                    imMayMove = false;
                }
            }

            if (imMayMove) {
                moveInputMethodWindowsIfNeededLocked(false);
            }

            mLayersController.assignLayersLocked(displayContent.getWindowList());
            // Don't do layout here, the window must call
            // relayout to be displayed, so we'll do it there.

            if (focusChanged) {
                mInputMonitor.setInputFocusLw(mCurrentFocus, false /*updateInputWindows*/);
            }
            mInputMonitor.updateInputWindowsLw(false /*force*/);

            if (localLOGV || DEBUG_ADD_REMOVE) Slog.v(TAG_WM, "addWindow: New client "
                    + client.asBinder() + ": window=" + win + " Callers=" + Debug.getCallers(5));

            if (win.isVisibleOrAdding() && updateOrientationFromAppTokensLocked(false)) {
                reportNewConfig = true;
            }
        }

        if (reportNewConfig) {
            sendNewConfiguration();
        }

        Binder.restoreCallingIdentity(origId);
        return res;
}
```

```java
    public int relayoutWindow(Session session, IWindow client, int seq,
            WindowManager.LayoutParams attrs, int requestedWidth,
            int requestedHeight, int viewVisibility, int flags,
            Rect outFrame, Rect outOverscanInsets, Rect outContentInsets,
            Rect outVisibleInsets, Rect outStableInsets, Rect outOutsets, Rect outBackdropFrame,
            Configuration outConfig, Surface outSurface) {
        int result = 0;
        boolean configChanged;
        boolean hasStatusBarPermission =
                mContext.checkCallingOrSelfPermission(android.Manifest.permission.STATUS_BAR)
                        == PackageManager.PERMISSION_GRANTED;

        long origId = Binder.clearCallingIdentity();
        synchronized(mWindowMap) {
            WindowState win = windowForClientLocked(session, client, false);
            if (win == null) {
                return 0;
            }

            WindowStateAnimator winAnimator = win.mWinAnimator;
            if (viewVisibility != View.GONE) {
                win.setRequestedSize(requestedWidth, requestedHeight);
            }

            int attrChanges = 0;
            int flagChanges = 0;
            if (attrs != null) {
                mPolicy.adjustWindowParamsLw(attrs);
                // if they don't have the permission, mask out the status bar bits
                if (seq == win.mSeq) {
                    int systemUiVisibility = attrs.systemUiVisibility
                            | attrs.subtreeSystemUiVisibility;
                    if ((systemUiVisibility & DISABLE_MASK) != 0) {
                        if (!hasStatusBarPermission) {
                            systemUiVisibility &= ~DISABLE_MASK;
                        }
                    }
                    win.mSystemUiVisibility = systemUiVisibility;
                }
                if (win.mAttrs.type != attrs.type) {
                    throw new IllegalArgumentException(
                            "Window type can not be changed after the window is added.");
                }

                // Odd choice but less odd than embedding in copyFrom()
                if ((attrs.privateFlags & WindowManager.LayoutParams.PRIVATE_FLAG_PRESERVE_GEOMETRY)
                        != 0) {
                    attrs.x = win.mAttrs.x;
                    attrs.y = win.mAttrs.y;
                    attrs.width = win.mAttrs.width;
                    attrs.height = win.mAttrs.height;
                }

                flagChanges = win.mAttrs.flags ^= attrs.flags;
                attrChanges = win.mAttrs.copyFrom(attrs);
                if ((attrChanges & (WindowManager.LayoutParams.LAYOUT_CHANGED
                        | WindowManager.LayoutParams.SYSTEM_UI_VISIBILITY_CHANGED)) != 0) {
                    win.mLayoutNeeded = true;
                }
            }

            if (DEBUG_LAYOUT) Slog.v(TAG_WM, "Relayout " + win + ": viewVisibility=" + viewVisibility
                    + " req=" + requestedWidth + "x" + requestedHeight + " " + win.mAttrs);
            winAnimator.mSurfaceDestroyDeferred = (flags & RELAYOUT_DEFER_SURFACE_DESTROY) != 0;
            win.mEnforceSizeCompat =
                    (win.mAttrs.privateFlags & PRIVATE_FLAG_COMPATIBLE_WINDOW) != 0;
            if ((attrChanges & WindowManager.LayoutParams.ALPHA_CHANGED) != 0) {
                winAnimator.mAlpha = attrs.alpha;
            }
            win.setWindowScale(win.mRequestedWidth, win.mRequestedHeight);

            if (win.mAttrs.surfaceInsets.left != 0
                    || win.mAttrs.surfaceInsets.top != 0
                    || win.mAttrs.surfaceInsets.right != 0
                    || win.mAttrs.surfaceInsets.bottom != 0) {
                winAnimator.setOpaqueLocked(false);
            }

            boolean imMayMove = (flagChanges & (FLAG_ALT_FOCUSABLE_IM | FLAG_NOT_FOCUSABLE)) != 0;
            final boolean isDefaultDisplay = win.isDefaultDisplay();
            boolean focusMayChange = isDefaultDisplay && (win.mViewVisibility != viewVisibility
                    || ((flagChanges & FLAG_NOT_FOCUSABLE) != 0)
                    || (!win.mRelayoutCalled));

            boolean wallpaperMayMove = win.mViewVisibility != viewVisibility
                    && (win.mAttrs.flags & FLAG_SHOW_WALLPAPER) != 0;
            wallpaperMayMove |= (flagChanges & FLAG_SHOW_WALLPAPER) != 0;
            if ((flagChanges & FLAG_SECURE) != 0 && winAnimator.mSurfaceController != null) {
                winAnimator.mSurfaceController.setSecure(isSecureLocked(win));
            }

            win.mRelayoutCalled = true;
            win.mInRelayout = true;

            final int oldVisibility = win.mViewVisibility;
            win.mViewVisibility = viewVisibility;
            if (DEBUG_SCREEN_ON) {
                RuntimeException stack = new RuntimeException();
                stack.fillInStackTrace();
                Slog.i(TAG_WM, "Relayout " + win + ": oldVis=" + oldVisibility
                        + " newVis=" + viewVisibility, stack);
            }
            if (viewVisibility == View.VISIBLE &&
                    (win.mAppToken == null || !win.mAppToken.clientHidden)) {
                result = relayoutVisibleWindow(outConfig, result, win, winAnimator, attrChanges,
                        oldVisibility);
                try {
                    result = createSurfaceControl(outSurface, result, win, winAnimator);
                } catch (Exception e) {
                    mInputMonitor.updateInputWindowsLw(true /*force*/);

                    Slog.w(TAG_WM, "Exception thrown when creating surface for client "
                             + client + " (" + win.mAttrs.getTitle() + ")",
                             e);
                    Binder.restoreCallingIdentity(origId);
                    return 0;
                }
                if ((result & WindowManagerGlobal.RELAYOUT_RES_FIRST_TIME) != 0) {
                    focusMayChange = isDefaultDisplay;
                }
                if (win.mAttrs.type == TYPE_INPUT_METHOD && mInputMethodWindow == null) {
                    mInputMethodWindow = win;
                    imMayMove = true;
                }
                win.adjustStartingWindowFlags();
            } else {
                winAnimator.mEnterAnimationPending = false;
                winAnimator.mEnteringAnimation = false;
                final boolean usingSavedSurfaceBeforeVisible =
                        oldVisibility != View.VISIBLE && win.isAnimatingWithSavedSurface();
                if (DEBUG_APP_TRANSITIONS || DEBUG_ANIM) {
                    if (winAnimator.hasSurface() && !win.mAnimatingExit
                            && usingSavedSurfaceBeforeVisible) {
                        Slog.d(TAG, "Ignoring layout to invisible when using saved surface " + win);
                    }
                }

                if (winAnimator.hasSurface() && !win.mAnimatingExit
                        && !usingSavedSurfaceBeforeVisible) {
                    if (DEBUG_VISIBILITY) Slog.i(TAG_WM, "Relayout invis " + win
                            + ": mAnimatingExit=" + win.mAnimatingExit);
                    // If we are not currently running the exit animation, we
                    // need to see about starting one.
                    // We don't want to animate visibility of windows which are pending
                    // replacement. In the case of activity relaunch child windows
                    // could request visibility changes as they are detached from the main
                    // application window during the tear down process. If we satisfied
                    // these visibility changes though, we would cause a visual glitch
                    // hiding the window before it's replacement was available.
                    // So we just do nothing on our side.
                    if (!win.mWillReplaceWindow) {
                        focusMayChange = tryStartExitingAnimation(
                                win, winAnimator, isDefaultDisplay, focusMayChange);
                    }
                    result |= RELAYOUT_RES_SURFACE_CHANGED;
                }
                if (viewVisibility == View.VISIBLE && winAnimator.hasSurface()) {
                    // We already told the client to go invisible, but the message may not be
                    // handled yet, or it might want to draw a last frame. If we already have a
                    // surface, let the client use that, but don't create new surface at this point.
                    winAnimator.mSurfaceController.getSurface(outSurface);
                } else {
                    if (DEBUG_VISIBILITY) Slog.i(TAG_WM, "Releasing surface in: " + win);

                    try {
                        Trace.traceBegin(Trace.TRACE_TAG_WINDOW_MANAGER, "wmReleaseOutSurface_"
                                + win.mAttrs.getTitle());
                        outSurface.release();
                    } finally {
                        Trace.traceEnd(Trace.TRACE_TAG_WINDOW_MANAGER);
                    }
                }
            }

            if (focusMayChange) {
                //System.out.println("Focus may change: " + win.mAttrs.getTitle());
                if (updateFocusedWindowLocked(UPDATE_FOCUS_WILL_PLACE_SURFACES,
                        false /*updateInputWindows*/)) {
                    imMayMove = false;
                }
                //System.out.println("Relayout " + win + ": focus=" + mCurrentFocus);
            }

            // updateFocusedWindowLocked() already assigned layers so we only need to
            // reassign them at this point if the IM window state gets shuffled
            boolean toBeDisplayed = (result & WindowManagerGlobal.RELAYOUT_RES_FIRST_TIME) != 0;
            if (imMayMove && (moveInputMethodWindowsIfNeededLocked(false) || toBeDisplayed)) {
                // Little hack here -- we -should- be able to rely on the
                // function to return true if the IME has moved and needs
                // its layer recomputed.  However, if the IME was hidden
                // and isn't actually moved in the list, its layer may be
                // out of data so we make sure to recompute it.
                mLayersController.assignLayersLocked(win.getWindowList());
            }

            if (wallpaperMayMove) {
                getDefaultDisplayContentLocked().pendingLayoutChanges |=
                        WindowManagerPolicy.FINISH_LAYOUT_REDO_WALLPAPER;
            }

            win.setDisplayLayoutNeeded();
            win.mGivenInsetsPending = (flags&WindowManagerGlobal.RELAYOUT_INSETS_PENDING) != 0;
            configChanged = updateOrientationFromAppTokensLocked(false);
            mWindowPlacerLocked.performSurfacePlacement();
            if (toBeDisplayed && win.mIsWallpaper) {
                DisplayInfo displayInfo = getDefaultDisplayInfoLocked();
                mWallpaperControllerLocked.updateWallpaperOffset(
                        win, displayInfo.logicalWidth, displayInfo.logicalHeight, false);
            }
            if (win.mAppToken != null) {
                win.mAppToken.updateReportedVisibilityLocked();
            }
            if (winAnimator.mReportSurfaceResized) {
                winAnimator.mReportSurfaceResized = false;
                result |= WindowManagerGlobal.RELAYOUT_RES_SURFACE_RESIZED;
            }
            if (mPolicy.isNavBarForcedShownLw(win)) {
                result |= WindowManagerGlobal.RELAYOUT_RES_CONSUME_ALWAYS_NAV_BAR;
            }
            if (!win.isGoneForLayoutLw()) {
                win.mResizedWhileGone = false;
            }
            outFrame.set(win.mCompatFrame);
            outOverscanInsets.set(win.mOverscanInsets);
            outContentInsets.set(win.mContentInsets);
            outVisibleInsets.set(win.mVisibleInsets);
            outStableInsets.set(win.mStableInsets);
            outOutsets.set(win.mOutsets);
            outBackdropFrame.set(win.getBackdropFrame(win.mFrame));
            if (localLOGV) Slog.v(
                TAG_WM, "Relayout given client " + client.asBinder()
                + ", requestedWidth=" + requestedWidth
                + ", requestedHeight=" + requestedHeight
                + ", viewVisibility=" + viewVisibility
                + "\nRelayout returning frame=" + outFrame
                + ", surface=" + outSurface);

            if (localLOGV || DEBUG_FOCUS) Slog.v(
                TAG_WM, "Relayout of " + win + ": focusMayChange=" + focusMayChange);

            result |= mInTouchMode ? WindowManagerGlobal.RELAYOUT_RES_IN_TOUCH_MODE : 0;

            mInputMonitor.updateInputWindowsLw(true /*force*/);

            if (DEBUG_LAYOUT) {
                Slog.v(TAG_WM, "Relayout complete " + win + ": outFrame=" + outFrame.toShortString());
            }
            win.mInRelayout = false;
        }

        if (configChanged) {
            sendNewConfiguration();
        }
        Binder.restoreCallingIdentity(origId);
        return result;
    }
```



## Session

```java
//frameworks/base/services/core/java/com/android/server/wm/Session.java
@Override
public int addToDisplayWithoutInputChannel(IWindow window, int seq, WindowManager.LayoutParams attrs,
            int viewVisibility, int displayId, Rect outContentInsets, Rect outStableInsets) {
      //调用WindowManagerService的addWindow方法
    return mService.addWindow(this, window, seq, attrs, viewVisibility, displayId,
        outContentInsets, outStableInsets, null /* outOutsets */, null);
}

```

## ViewManager

```java
//frameworks/base/core/java/android/view/ViewManager.java
public interface ViewManager
{
    /**
     * Assign the passed LayoutParams to the passed View and add the view to the window.
     * <p>Throws {@link android.view.WindowManager.BadTokenException} for certain programming
     * errors, such as adding a second view to a window without removing the first view.
     * <p>Throws {@link android.view.WindowManager.InvalidDisplayException} if the window is on a
     * secondary {@link Display} and the specified display can't be found
     * (see {@link android.app.Presentation}).
     * @param view The view to be added to this window.
     * @param params The LayoutParams to assign to view.
     */
    public void addView(View view, ViewGroup.LayoutParams params);
    public void updateViewLayout(View view, ViewGroup.LayoutParams params);
    public void removeView(View view);
}
```

## WindowManager

```java
//frameworks/base/core/java/android/view/WindowManager.java
```



```java
//frameworks/base/core/java/android/view/WindowManagerImpl.java
```

## WindowManagerGlobal

### getWindowSession

```java
public static IWindowSession getWindowSession() {
    synchronized (WindowManagerGlobal.class) {
        if (sWindowSession == null) {
            try {
                InputMethodManager imm = InputMethodManager.getInstance();
                IWindowManager windowManager = getWindowManagerService(); //获取WindowManagerService
                sWindowSession = windowManager.openSession(
                        new IWindowSessionCallback.Stub() {
                            @Override
                            public void onAnimatorScaleChanged(float scale) {
                                ValueAnimator.setDurationScale(scale);
                            }
                        },
                        imm.getClient(), imm.getInputContext());
            } catch (RemoteException e) {
                throw e.rethrowFromSystemServer();
            }
        }
        return sWindowSession;
    }
}
```

### getWindowManagerService

```java
public static IWindowManager getWindowManagerService() {
    synchronized (WindowManagerGlobal.class) {
        if (sWindowManagerService == null) {
            sWindowManagerService = IWindowManager.Stub.asInterface(
                    ServiceManager.getService("window"));
            try {
                sWindowManagerService = getWindowManagerService();
              ValueAnimator.setDurationScale(sWindowManagerService.getCurrentAnimatorScale());
            } catch (RemoteException e) {
                throw e.rethrowFromSystemServer();
            }
        }
        return sWindowManagerService;
    }
}
```





```java
//frameworks/base/core/java/android/view/WindowManagerGlobal.java
```

## ViewRootImpl

### 构造函数

```java
//frameworks/base/core/java/android/view/ViewRootImpl.java

final Surface mSurface = new Surface();
public ViewRootImpl(Context context, Display display) {
    mContext = context;
    mWindowSession = WindowManagerGlobal.getWindowSession();//获取WindowSession
    mDisplay = display;
    mBasePackageName = context.getBasePackageName();
    mThread = Thread.currentThread();
    mLocation = new WindowLeaked(null);
    mLocation.fillInStackTrace();
    mWidth = -1;
    mHeight = -1;
    mDirty = new Rect();
    mTempRect = new Rect();
    mVisRect = new Rect();
    mWinFrame = new Rect();
    mWindow = new W(this);
    mTargetSdkVersion = context.getApplicationInfo().targetSdkVersion;
    mViewVisibility = View.GONE;
    mTransparentRegion = new Region();
    mPreviousTransparentRegion = new Region();
    mFirst = true; // true for the first time the view is added
    mAdded = false;
    mAttachInfo = new View.AttachInfo(mWindowSession, mWindow, display, this, mHandler, this);
    mAccessibilityManager = AccessibilityManager.getInstance(context);
    mAccessibilityInteractionConnectionManager =
        new AccessibilityInteractionConnectionManager();
    mAccessibilityManager.addAccessibilityStateChangeListener(
            mAccessibilityInteractionConnectionManager);
    mHighContrastTextManager = new HighContrastTextManager();
    mAccessibilityManager.addHighTextContrastStateChangeListener(
            mHighContrastTextManager);
    mViewConfiguration = ViewConfiguration.get(context);
    mDensity = context.getResources().getDisplayMetrics().densityDpi;
    mNoncompatDensity = context.getResources().getDisplayMetrics().noncompatDensityDpi;
    mFallbackEventHandler = new PhoneFallbackEventHandler(context);
    mChoreographer = Choreographer.getInstance(); //获取Choreographer实例
    mDisplayManager = (DisplayManager)context.getSystemService(Context.DISPLAY_SERVICE);
    loadSystemProperties();
}
```



```java
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
  synchronized (this) {
      if (mView == null) {
          mView = view;

          mAttachInfo.mDisplayState = mDisplay.getState();
          mDisplayManager.registerDisplayListener(mDisplayListener, mHandler);

          mViewLayoutDirectionInitial = mView.getRawLayoutDirection();
          mFallbackEventHandler.setView(view);
          mWindowAttributes.copyFrom(attrs);
          if (mWindowAttributes.packageName == null) {
              mWindowAttributes.packageName = mBasePackageName;
          }
          attrs = mWindowAttributes;
          setTag();

          if (DEBUG_KEEP_SCREEN_ON && (mClientWindowLayoutFlags
                  & WindowManager.LayoutParams.FLAG_KEEP_SCREEN_ON) != 0
                  && (attrs.flags&WindowManager.LayoutParams.FLAG_KEEP_SCREEN_ON) == 0) {
              Slog.d(mTag, "setView: FLAG_KEEP_SCREEN_ON changed from true to false!");
          }
          // Keep track of the actual window flags supplied by the client.
          mClientWindowLayoutFlags = attrs.flags;

          setAccessibilityFocus(null, null);

          if (view instanceof RootViewSurfaceTaker) {
              mSurfaceHolderCallback =
                      ((RootViewSurfaceTaker)view).willYouTakeTheSurface();
              if (mSurfaceHolderCallback != null) {
                  mSurfaceHolder = new TakenSurfaceHolder();
                  mSurfaceHolder.setFormat(PixelFormat.UNKNOWN);
              }
          }

          // Compute surface insets required to draw at specified Z value.
          // TODO: Use real shadow insets for a constant max Z.
          if (!attrs.hasManualSurfaceInsets) {
              attrs.setSurfaceInsets(view, false /*manual*/, true /*preservePrevious*/);
          }

          CompatibilityInfo compatibilityInfo =
                  mDisplay.getDisplayAdjustments().getCompatibilityInfo();
          mTranslator = compatibilityInfo.getTranslator();

          // If the application owns the surface, don't enable hardware acceleration
          if (mSurfaceHolder == null) {
              enableHardwareAcceleration(attrs);
          }

          boolean restore = false;
          if (mTranslator != null) {
              mSurface.setCompatibilityTranslator(mTranslator);
              restore = true;
              attrs.backup();
              mTranslator.translateWindowLayout(attrs);
          }
          if (DEBUG_LAYOUT) Log.d(mTag, "WindowLayout in setView:" + attrs);

          if (!compatibilityInfo.supportsScreen()) {
              attrs.privateFlags |= WindowManager.LayoutParams.PRIVATE_FLAG_COMPATIBLE_WINDOW;
              mLastInCompatMode = true;
          }

          mSoftInputMode = attrs.softInputMode;
          mWindowAttributesChanged = true;
          mWindowAttributesChangesFlag = WindowManager.LayoutParams.EVERYTHING_CHANGED;
          mAttachInfo.mRootView = view;
          mAttachInfo.mScalingRequired = mTranslator != null;
          mAttachInfo.mApplicationScale =
                  mTranslator == null ? 1.0f : mTranslator.applicationScale;
          if (panelParentView != null) {
              mAttachInfo.mPanelParentWindowToken
                      = panelParentView.getApplicationWindowToken();
          }
          mAdded = true;
          int res; /* = WindowManagerImpl.ADD_OKAY; */

          // Schedule the first layout -before- adding to the window
          // manager, to make sure we do the relayout before receiving
          // any other events from the system.
          requestLayout();
          if ((mWindowAttributes.inputFeatures
                  & WindowManager.LayoutParams.INPUT_FEATURE_NO_INPUT_CHANNEL) == 0) {
              mInputChannel = new InputChannel();
          }
          mForceDecorViewVisibility = (mWindowAttributes.privateFlags
                  & PRIVATE_FLAG_FORCE_DECOR_VIEW_VISIBILITY) != 0;
          try {
              mOrigWindowType = mWindowAttributes.type;
              mAttachInfo.mRecomputeGlobalAttributes = true;
              collectViewAttributes();
              //调用mWindowSession的addToDisplay方法
              res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                      getHostVisibility(), mDisplay.getDisplayId(),
                      mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                      mAttachInfo.mOutsets, mInputChannel);
          } catch (RemoteException e) {
              mAdded = false;
              mView = null;
              mAttachInfo.mRootView = null;
              mInputChannel = null;
              mFallbackEventHandler.setView(null);
              unscheduleTraversals();
              setAccessibilityFocus(null, null);
              throw new RuntimeException("Adding window failed", e);
          } finally {
              if (restore) {
                  attrs.restore();
              }
          }

          if (mTranslator != null) {
              mTranslator.translateRectInScreenToAppWindow(mAttachInfo.mContentInsets);
          }
          mPendingOverscanInsets.set(0, 0, 0, 0);
          mPendingContentInsets.set(mAttachInfo.mContentInsets);
          mPendingStableInsets.set(mAttachInfo.mStableInsets);
          mPendingVisibleInsets.set(0, 0, 0, 0);
          mAttachInfo.mAlwaysConsumeNavBar =
                  (res & WindowManagerGlobal.ADD_FLAG_ALWAYS_CONSUME_NAV_BAR) != 0;
          mPendingAlwaysConsumeNavBar = mAttachInfo.mAlwaysConsumeNavBar;
          if (DEBUG_LAYOUT) Log.v(mTag, "Added window " + mWindow);
          if (res < WindowManagerGlobal.ADD_OKAY) {
              mAttachInfo.mRootView = null;
              mAdded = false;
              mFallbackEventHandler.setView(null);
              unscheduleTraversals();
              setAccessibilityFocus(null, null);
              switch (res) {
                  case WindowManagerGlobal.ADD_BAD_APP_TOKEN:
                  case WindowManagerGlobal.ADD_BAD_SUBWINDOW_TOKEN:
                      throw new WindowManager.BadTokenException(
                              "Unable to add window -- token " + attrs.token
                              + " is not valid; is your activity running?");
                  case WindowManagerGlobal.ADD_NOT_APP_TOKEN:
                      throw new WindowManager.BadTokenException(
                              "Unable to add window -- token " + attrs.token
                              + " is not for an application");
                  case WindowManagerGlobal.ADD_APP_EXITING:
                      throw new WindowManager.BadTokenException(
                              "Unable to add window -- app for token " + attrs.token
                              + " is exiting");
                  case WindowManagerGlobal.ADD_DUPLICATE_ADD:
                      throw new WindowManager.BadTokenException(
                              "Unable to add window -- window " + mWindow
                              + " has already been added");
                  case WindowManagerGlobal.ADD_STARTING_NOT_NEEDED:
                      // Silently ignore -- we would have just removed it
                      // right away, anyway.
                      return;
                  case WindowManagerGlobal.ADD_MULTIPLE_SINGLETON:
                      throw new WindowManager.BadTokenException("Unable to add window "
                              + mWindow + " -- another window of type "
                              + mWindowAttributes.type + " already exists");
                  case WindowManagerGlobal.ADD_PERMISSION_DENIED:
                      throw new WindowManager.BadTokenException("Unable to add window "
                              + mWindow + " -- permission denied for window type "
                              + mWindowAttributes.type);
                  case WindowManagerGlobal.ADD_INVALID_DISPLAY:
                      throw new WindowManager.InvalidDisplayException("Unable to add window "
                              + mWindow + " -- the specified display can not be found");
                  case WindowManagerGlobal.ADD_INVALID_TYPE:
                      throw new WindowManager.InvalidDisplayException("Unable to add window "
                              + mWindow + " -- the specified window type "
                              + mWindowAttributes.type + " is not valid");
              }
              throw new RuntimeException(
                      "Unable to add window -- unknown error code " + res);
          }

          if (view instanceof RootViewSurfaceTaker) {
              mInputQueueCallback =
                  ((RootViewSurfaceTaker)view).willYouTakeTheInputQueue();
          }
          if (mInputChannel != null) {
              if (mInputQueueCallback != null) {
                  mInputQueue = new InputQueue();
                  mInputQueueCallback.onInputQueueCreated(mInputQueue);
              }
              mInputEventReceiver = new WindowInputEventReceiver(mInputChannel,
                      Looper.myLooper());
          }

          view.assignParent(this);
          mAddedTouchMode = (res & WindowManagerGlobal.ADD_FLAG_IN_TOUCH_MODE) != 0;
          mAppVisible = (res & WindowManagerGlobal.ADD_FLAG_APP_VISIBLE) != 0;

          if (mAccessibilityManager.isEnabled()) {
              mAccessibilityInteractionConnectionManager.ensureConnection();
          }

          if (view.getImportantForAccessibility() == View.IMPORTANT_FOR_ACCESSIBILITY_AUTO) {
              view.setImportantForAccessibility(View.IMPORTANT_FOR_ACCESSIBILITY_YES);
          }

          // Set up the input pipeline.
          CharSequence counterSuffix = attrs.getTitle();
          mSyntheticInputStage = new SyntheticInputStage();
          InputStage viewPostImeStage = new ViewPostImeInputStage(mSyntheticInputStage);
          InputStage nativePostImeStage = new NativePostImeInputStage(viewPostImeStage,
                  "aq:native-post-ime:" + counterSuffix);
          InputStage earlyPostImeStage = new EarlyPostImeInputStage(nativePostImeStage);
          InputStage imeStage = new ImeInputStage(earlyPostImeStage,
                  "aq:ime:" + counterSuffix);
          InputStage viewPreImeStage = new ViewPreImeInputStage(imeStage);
          InputStage nativePreImeStage = new NativePreImeInputStage(viewPreImeStage,
                  "aq:native-pre-ime:" + counterSuffix);

          mFirstInputStage = nativePreImeStage;
          mFirstPostImeInputStage = earlyPostImeStage;
          mPendingInputEventQueueLengthCounterName = "aq:pending:" + counterSuffix;
      }
  }
}
```



```java
 private int relayoutWindow(WindowManager.LayoutParams params, int viewVisibility,
            boolean insetsPending) throws RemoteException {
    float appScale = mAttachInfo.mApplicationScale;
    boolean restore = false;
    if (params != null && mTranslator != null) {
        restore = true;
        params.backup();
        mTranslator.translateWindowLayout(params);
    }
    if (params != null) {
        if (DBG) Log.d(mTag, "WindowLayout in layoutWindow:" + params);
    }
    mPendingConfiguration.seq = 0;
    //Log.d(mTag, ">>>>>> CALLING relayout");
    if (params != null && mOrigWindowType != params.type) {
        // For compatibility with old apps, don't crash here.
        if (mTargetSdkVersion < Build.VERSION_CODES.ICE_CREAM_SANDWICH) {
            Slog.w(mTag, "Window type can not be changed after "
                    + "the window is added; ignoring change of " + mView);
            params.type = mOrigWindowType;
        }
    }
    int relayoutResult = mWindowSession.relayout(
            mWindow, mSeq, params,
            (int) (mView.getMeasuredWidth() * appScale + 0.5f),
            (int) (mView.getMeasuredHeight() * appScale + 0.5f),
            viewVisibility, insetsPending ? WindowManagerGlobal.RELAYOUT_INSETS_PENDING : 0,
            mWinFrame, mPendingOverscanInsets, mPendingContentInsets, mPendingVisibleInsets,
            mPendingStableInsets, mPendingOutsets, mPendingBackDropFrame, mPendingConfiguration,
            mSurface);

    mPendingAlwaysConsumeNavBar =
            (relayoutResult & WindowManagerGlobal.RELAYOUT_RES_CONSUME_ALWAYS_NAV_BAR) != 0;

    //Log.d(mTag, "<<<<<< BACK FROM relayout");
    if (restore) {
        params.restore();
    }

    if (mTranslator != null) {
        mTranslator.translateRectInScreenToAppWinFrame(mWinFrame);
        mTranslator.translateRectInScreenToAppWindow(mPendingOverscanInsets);
        mTranslator.translateRectInScreenToAppWindow(mPendingContentInsets);
        mTranslator.translateRectInScreenToAppWindow(mPendingVisibleInsets);
        mTranslator.translateRectInScreenToAppWindow(mPendingStableInsets);
    }
    return relayoutResult;
}
```

```java
void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;//避免重复调用
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier(); //发送消息屏障
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);//发送消息
        notifyRendererOfFramePending();
        pokeDrawLockIfNeeded();
    }
}
```



## 参考

* [Android 显示系统：SurfaceFlinger详解](https://www.cnblogs.com/blogs-of-lxl/p/11272756.html)