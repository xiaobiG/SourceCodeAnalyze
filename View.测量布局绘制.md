# ViewRootImpl

ViewRootImpl充当的是View和window之间的纽带。在startActivity之后，经过与ActivityManagerService的IPC交互，会在ActivityThread的handleResumeActivity方法中执行到getWindow().addView，就是将根布局 Decor添加到window中以显示。getWindow会以WindowManagerGloble来执行addView方法，其中就会创建ViewRootImpl实例并调用其setView方法传入Decor布局，在setView中会执行到performTranvesals方法，这个方法是重点：

```java

1676    private void performTraversals() {
1677        // cache mView since it is used so much below...
1678        final View host = mView;
1679
1680        if (DBG) {
1681            System.out.println("======================================");
1682            System.out.println("performTraversals");
1683            host.debug();
1684        }
1685
1686        if (host == null || !mAdded)
1687            return;
1688
1689        mIsInTraversal = true;
1690        mWillDrawSoon = true;
1691        boolean windowSizeMayChange = false;
1692        boolean newSurface = false;
1693        boolean surfaceChanged = false;
1694        WindowManager.LayoutParams lp = mWindowAttributes;
1695
1696        int desiredWindowWidth;
1697        int desiredWindowHeight;
1698
1699        final int viewVisibility = getHostVisibility();
1700        final boolean viewVisibilityChanged = !mFirst
1701                && (mViewVisibility != viewVisibility || mNewSurfaceNeeded
1702                // Also check for possible double visibility update, which will make current
1703                // viewVisibility value equal to mViewVisibility and we may miss it.
1704                || mAppVisibilityChanged);
1705        mAppVisibilityChanged = false;
1706        final boolean viewUserVisibilityChanged = !mFirst &&
1707                ((mViewVisibility == View.VISIBLE) != (viewVisibility == View.VISIBLE));
1708
1709        WindowManager.LayoutParams params = null;
1710        if (mWindowAttributesChanged) {
1711            mWindowAttributesChanged = false;
1712            surfaceChanged = true;
1713            params = lp;
1714        }
1715        CompatibilityInfo compatibilityInfo =
1716                mDisplay.getDisplayAdjustments().getCompatibilityInfo();
1717        if (compatibilityInfo.supportsScreen() == mLastInCompatMode) {
1718            params = lp;
1719            mFullRedrawNeeded = true;
1720            mLayoutRequested = true;
1721            if (mLastInCompatMode) {
1722                params.privateFlags &= ~WindowManager.LayoutParams.PRIVATE_FLAG_COMPATIBLE_WINDOW;
1723                mLastInCompatMode = false;
1724            } else {
1725                params.privateFlags |= WindowManager.LayoutParams.PRIVATE_FLAG_COMPATIBLE_WINDOW;
1726                mLastInCompatMode = true;
1727            }
1728        }
1729
1730        mWindowAttributesChangesFlag = 0;
1731
1732        Rect frame = mWinFrame;
1733        if (mFirst) {
1734            mFullRedrawNeeded = true;
1735            mLayoutRequested = true;
1736
1737            final Configuration config = mContext.getResources().getConfiguration();
1738            if (shouldUseDisplaySize(lp)) {
1739                // NOTE -- system code, won't try to do compat mode.
1740                Point size = new Point();
1741                mDisplay.getRealSize(size);
1742                desiredWindowWidth = size.x;
1743                desiredWindowHeight = size.y;
1744            } else {
1745                desiredWindowWidth = mWinFrame.width();
1746                desiredWindowHeight = mWinFrame.height();
1747            }
1748
1749            // We used to use the following condition to choose 32 bits drawing caches:
1750            // PixelFormat.hasAlpha(lp.format) || lp.format == PixelFormat.RGBX_8888
1751            // However, windows are now always 32 bits by default, so choose 32 bits
1752            mAttachInfo.mUse32BitDrawingCache = true;
1753            mAttachInfo.mHasWindowFocus = false;
1754            mAttachInfo.mWindowVisibility = viewVisibility;
1755            mAttachInfo.mRecomputeGlobalAttributes = false;
1756            mLastConfigurationFromResources.setTo(config);
1757            mLastSystemUiVisibility = mAttachInfo.mSystemUiVisibility;
1758            // Set the layout direction if it has not been set before (inherit is the default)
1759            if (mViewLayoutDirectionInitial == View.LAYOUT_DIRECTION_INHERIT) {
1760                host.setLayoutDirection(config.getLayoutDirection());
1761            }
1762            host.dispatchAttachedToWindow(mAttachInfo, 0);
1763            mAttachInfo.mTreeObserver.dispatchOnWindowAttachedChange(true);
1764            dispatchApplyInsets(host);
1765        } else {
1766            desiredWindowWidth = frame.width();
1767            desiredWindowHeight = frame.height();
1768            if (desiredWindowWidth != mWidth || desiredWindowHeight != mHeight) {
1769                if (DEBUG_ORIENTATION) Log.v(mTag, "View " + host + " resized to: " + frame);
1770                mFullRedrawNeeded = true;
1771                mLayoutRequested = true;
1772                windowSizeMayChange = true;
1773            }
1774        }
1775
1776        if (viewVisibilityChanged) {
1777            mAttachInfo.mWindowVisibility = viewVisibility;
1778            host.dispatchWindowVisibilityChanged(viewVisibility);
1779            if (viewUserVisibilityChanged) {
1780                host.dispatchVisibilityAggregated(viewVisibility == View.VISIBLE);
1781            }
1782            if (viewVisibility != View.VISIBLE || mNewSurfaceNeeded) {
1783                endDragResizing();
1784                destroyHardwareResources();
1785            }
1786            if (viewVisibility == View.GONE) {
1787                // After making a window gone, we will count it as being
1788                // shown for the first time the next time it gets focus.
1789                mHasHadWindowFocus = false;
1790            }
1791        }
1792
1793        // Non-visible windows can't hold accessibility focus.
1794        if (mAttachInfo.mWindowVisibility != View.VISIBLE) {
1795            host.clearAccessibilityFocus();
1796        }
1797
1798        // Execute enqueued actions on every traversal in case a detached view enqueued an action
1799        getRunQueue().executeActions(mAttachInfo.mHandler);
1800
1801        boolean insetsChanged = false;
1802
1803        boolean layoutRequested = mLayoutRequested && (!mStopped || mReportNextDraw);
1804        if (layoutRequested) {
1805
1806            final Resources res = mView.getContext().getResources();
1807
1808            if (mFirst) {
1809                // make sure touch mode code executes by setting cached value
1810                // to opposite of the added touch mode.
1811                mAttachInfo.mInTouchMode = !mAddedTouchMode;
1812                ensureTouchModeLocally(mAddedTouchMode);
1813            } else {
1814                if (!mPendingOverscanInsets.equals(mAttachInfo.mOverscanInsets)) {
1815                    insetsChanged = true;
1816                }
1817                if (!mPendingContentInsets.equals(mAttachInfo.mContentInsets)) {
1818                    insetsChanged = true;
1819                }
1820                if (!mPendingStableInsets.equals(mAttachInfo.mStableInsets)) {
1821                    insetsChanged = true;
1822                }
1823                if (!mPendingDisplayCutout.equals(mAttachInfo.mDisplayCutout)) {
1824                    insetsChanged = true;
1825                }
1826                if (!mPendingVisibleInsets.equals(mAttachInfo.mVisibleInsets)) {
1827                    mAttachInfo.mVisibleInsets.set(mPendingVisibleInsets);
1828                    if (DEBUG_LAYOUT) Log.v(mTag, "Visible insets changing to: "
1829                            + mAttachInfo.mVisibleInsets);
1830                }
1831                if (!mPendingOutsets.equals(mAttachInfo.mOutsets)) {
1832                    insetsChanged = true;
1833                }
1834                if (mPendingAlwaysConsumeNavBar != mAttachInfo.mAlwaysConsumeNavBar) {
1835                    insetsChanged = true;
1836                }
1837                if (lp.width == ViewGroup.LayoutParams.WRAP_CONTENT
1838                        || lp.height == ViewGroup.LayoutParams.WRAP_CONTENT) {
1839                    windowSizeMayChange = true;
1840
1841                    if (shouldUseDisplaySize(lp)) {
1842                        // NOTE -- system code, won't try to do compat mode.
1843                        Point size = new Point();
1844                        mDisplay.getRealSize(size);
1845                        desiredWindowWidth = size.x;
1846                        desiredWindowHeight = size.y;
1847                    } else {
1848                        Configuration config = res.getConfiguration();
1849                        desiredWindowWidth = dipToPx(config.screenWidthDp);
1850                        desiredWindowHeight = dipToPx(config.screenHeightDp);
1851                    }
1852                }
1853            }
1854
1855            // Ask host how big it wants to be
1856            windowSizeMayChange |= measureHierarchy(host, lp, res,
1857                    desiredWindowWidth, desiredWindowHeight);
1858        }
1859
1860        if (collectViewAttributes()) {
1861            params = lp;
1862        }
1863        if (mAttachInfo.mForceReportNewAttributes) {
1864            mAttachInfo.mForceReportNewAttributes = false;
1865            params = lp;
1866        }
1867
1868        if (mFirst || mAttachInfo.mViewVisibilityChanged) {
1869            mAttachInfo.mViewVisibilityChanged = false;
1870            int resizeMode = mSoftInputMode &
1871                    WindowManager.LayoutParams.SOFT_INPUT_MASK_ADJUST;
1872            // If we are in auto resize mode, then we need to determine
1873            // what mode to use now.
1874            if (resizeMode == WindowManager.LayoutParams.SOFT_INPUT_ADJUST_UNSPECIFIED) {
1875                final int N = mAttachInfo.mScrollContainers.size();
1876                for (int i=0; i<N; i++) {
1877                    if (mAttachInfo.mScrollContainers.get(i).isShown()) {
1878                        resizeMode = WindowManager.LayoutParams.SOFT_INPUT_ADJUST_RESIZE;
1879                    }
1880                }
1881                if (resizeMode == 0) {
1882                    resizeMode = WindowManager.LayoutParams.SOFT_INPUT_ADJUST_PAN;
1883                }
1884                if ((lp.softInputMode &
1885                        WindowManager.LayoutParams.SOFT_INPUT_MASK_ADJUST) != resizeMode) {
1886                    lp.softInputMode = (lp.softInputMode &
1887                            ~WindowManager.LayoutParams.SOFT_INPUT_MASK_ADJUST) |
1888                            resizeMode;
1889                    params = lp;
1890                }
1891            }
1892        }
1893
1894        if (params != null) {
1895            if ((host.mPrivateFlags & View.PFLAG_REQUEST_TRANSPARENT_REGIONS) != 0) {
1896                if (!PixelFormat.formatHasAlpha(params.format)) {
1897                    params.format = PixelFormat.TRANSLUCENT;
1898                }
1899            }
1900            mAttachInfo.mOverscanRequested = (params.flags
1901                    & WindowManager.LayoutParams.FLAG_LAYOUT_IN_OVERSCAN) != 0;
1902        }
1903
1904        if (mApplyInsetsRequested) {
1905            mApplyInsetsRequested = false;
1906            mLastOverscanRequested = mAttachInfo.mOverscanRequested;
1907            dispatchApplyInsets(host);
1908            if (mLayoutRequested) {
1909                // Short-circuit catching a new layout request here, so
1910                // we don't need to go through two layout passes when things
1911                // change due to fitting system windows, which can happen a lot.
1912                windowSizeMayChange |= measureHierarchy(host, lp,
1913                        mView.getContext().getResources(),
1914                        desiredWindowWidth, desiredWindowHeight);
1915            }
1916        }
1917
1918        if (layoutRequested) {
1919            // Clear this now, so that if anything requests a layout in the
1920            // rest of this function we will catch it and re-run a full
1921            // layout pass.
1922            mLayoutRequested = false;
1923        }
1924
1925        boolean windowShouldResize = layoutRequested && windowSizeMayChange
1926            && ((mWidth != host.getMeasuredWidth() || mHeight != host.getMeasuredHeight())
1927                || (lp.width == ViewGroup.LayoutParams.WRAP_CONTENT &&
1928                        frame.width() < desiredWindowWidth && frame.width() != mWidth)
1929                || (lp.height == ViewGroup.LayoutParams.WRAP_CONTENT &&
1930                        frame.height() < desiredWindowHeight && frame.height() != mHeight));
1931        windowShouldResize |= mDragResizing && mResizeMode == RESIZE_MODE_FREEFORM;
1932
1933        // If the activity was just relaunched, it might have unfrozen the task bounds (while
1934        // relaunching), so we need to force a call into window manager to pick up the latest
1935        // bounds.
1936        windowShouldResize |= mActivityRelaunched;
1937
1938        // Determine whether to compute insets.
1939        // If there are no inset listeners remaining then we may still need to compute
1940        // insets in case the old insets were non-empty and must be reset.
1941        final boolean computesInternalInsets =
1942                mAttachInfo.mTreeObserver.hasComputeInternalInsetsListeners()
1943                || mAttachInfo.mHasNonEmptyGivenInternalInsets;
1944
1945        boolean insetsPending = false;
1946        int relayoutResult = 0;
1947        boolean updatedConfiguration = false;
1948
1949        final int surfaceGenerationId = mSurface.getGenerationId();
1950
1951        final boolean isViewVisible = viewVisibility == View.VISIBLE;
1952        final boolean windowRelayoutWasForced = mForceNextWindowRelayout;
1953        if (mFirst || windowShouldResize || insetsChanged ||
1954                viewVisibilityChanged || params != null || mForceNextWindowRelayout) {
1955            mForceNextWindowRelayout = false;
1956
1957            if (isViewVisible) {
1958                // If this window is giving internal insets to the window
1959                // manager, and it is being added or changing its visibility,
1960                // then we want to first give the window manager "fake"
1961                // insets to cause it to effectively ignore the content of
1962                // the window during layout.  This avoids it briefly causing
1963                // other windows to resize/move based on the raw frame of the
1964                // window, waiting until we can finish laying out this window
1965                // and get back to the window manager with the ultimately
1966                // computed insets.
1967                insetsPending = computesInternalInsets && (mFirst || viewVisibilityChanged);
1968            }
1969
1970            if (mSurfaceHolder != null) {
1971                mSurfaceHolder.mSurfaceLock.lock();
1972                mDrawingAllowed = true;
1973            }
1974
1975            boolean hwInitialized = false;
1976            boolean contentInsetsChanged = false;
1977            boolean hadSurface = mSurface.isValid();
1978
1979            try {
1980                if (DEBUG_LAYOUT) {
1981                    Log.i(mTag, "host=w:" + host.getMeasuredWidth() + ", h:" +
1982                            host.getMeasuredHeight() + ", params=" + params);
1983                }
1984
1985                if (mAttachInfo.mThreadedRenderer != null) {
1986                    // relayoutWindow may decide to destroy mSurface. As that decision
1987                    // happens in WindowManager service, we need to be defensive here
1988                    // and stop using the surface in case it gets destroyed.
1989                    if (mAttachInfo.mThreadedRenderer.pauseSurface(mSurface)) {
1990                        // Animations were running so we need to push a frame
1991                        // to resume them
1992                        mDirty.set(0, 0, mWidth, mHeight);
1993                    }
1994                    mChoreographer.mFrameInfo.addFlags(FrameInfo.FLAG_WINDOW_LAYOUT_CHANGED);
1995                }
1996                relayoutResult = relayoutWindow(params, viewVisibility, insetsPending);
1997
1998                if (DEBUG_LAYOUT) Log.v(mTag, "relayout: frame=" + frame.toShortString()
1999                        + " overscan=" + mPendingOverscanInsets.toShortString()
2000                        + " content=" + mPendingContentInsets.toShortString()
2001                        + " visible=" + mPendingVisibleInsets.toShortString()
2002                        + " stable=" + mPendingStableInsets.toShortString()
2003                        + " cutout=" + mPendingDisplayCutout.get().toString()
2004                        + " outsets=" + mPendingOutsets.toShortString()
2005                        + " surface=" + mSurface);
2006
2007                // If the pending {@link MergedConfiguration} handed back from
2008                // {@link #relayoutWindow} does not match the one last reported,
2009                // WindowManagerService has reported back a frame from a configuration not yet
2010                // handled by the client. In this case, we need to accept the configuration so we
2011                // do not lay out and draw with the wrong configuration.
2012                if (!mPendingMergedConfiguration.equals(mLastReportedMergedConfiguration)) {
2013                    if (DEBUG_CONFIGURATION) Log.v(mTag, "Visible with new config: "
2014                            + mPendingMergedConfiguration.getMergedConfiguration());
2015                    performConfigurationChange(mPendingMergedConfiguration, !mFirst,
2016                            INVALID_DISPLAY /* same display */);
2017                    updatedConfiguration = true;
2018                }
2019
2020                final boolean overscanInsetsChanged = !mPendingOverscanInsets.equals(
2021                        mAttachInfo.mOverscanInsets);
2022                contentInsetsChanged = !mPendingContentInsets.equals(
2023                        mAttachInfo.mContentInsets);
2024                final boolean visibleInsetsChanged = !mPendingVisibleInsets.equals(
2025                        mAttachInfo.mVisibleInsets);
2026                final boolean stableInsetsChanged = !mPendingStableInsets.equals(
2027                        mAttachInfo.mStableInsets);
2028                final boolean cutoutChanged = !mPendingDisplayCutout.equals(
2029                        mAttachInfo.mDisplayCutout);
2030                final boolean outsetsChanged = !mPendingOutsets.equals(mAttachInfo.mOutsets);
2031                final boolean surfaceSizeChanged = (relayoutResult
2032                        & WindowManagerGlobal.RELAYOUT_RES_SURFACE_RESIZED) != 0;
2033                surfaceChanged |= surfaceSizeChanged;
2034                final boolean alwaysConsumeNavBarChanged =
2035                        mPendingAlwaysConsumeNavBar != mAttachInfo.mAlwaysConsumeNavBar;
2036                if (contentInsetsChanged) {
2037                    mAttachInfo.mContentInsets.set(mPendingContentInsets);
2038                    if (DEBUG_LAYOUT) Log.v(mTag, "Content insets changing to: "
2039                            + mAttachInfo.mContentInsets);
2040                }
2041                if (overscanInsetsChanged) {
2042                    mAttachInfo.mOverscanInsets.set(mPendingOverscanInsets);
2043                    if (DEBUG_LAYOUT) Log.v(mTag, "Overscan insets changing to: "
2044                            + mAttachInfo.mOverscanInsets);
2045                    // Need to relayout with content insets.
2046                    contentInsetsChanged = true;
2047                }
2048                if (stableInsetsChanged) {
2049                    mAttachInfo.mStableInsets.set(mPendingStableInsets);
2050                    if (DEBUG_LAYOUT) Log.v(mTag, "Decor insets changing to: "
2051                            + mAttachInfo.mStableInsets);
2052                    // Need to relayout with content insets.
2053                    contentInsetsChanged = true;
2054                }
2055                if (cutoutChanged) {
2056                    mAttachInfo.mDisplayCutout.set(mPendingDisplayCutout);
2057                    if (DEBUG_LAYOUT) {
2058                        Log.v(mTag, "DisplayCutout changing to: " + mAttachInfo.mDisplayCutout);
2059                    }
2060                    // Need to relayout with content insets.
2061                    contentInsetsChanged = true;
2062                }
2063                if (alwaysConsumeNavBarChanged) {
2064                    mAttachInfo.mAlwaysConsumeNavBar = mPendingAlwaysConsumeNavBar;
2065                    contentInsetsChanged = true;
2066                }
2067                if (contentInsetsChanged || mLastSystemUiVisibility !=
2068                        mAttachInfo.mSystemUiVisibility || mApplyInsetsRequested
2069                        || mLastOverscanRequested != mAttachInfo.mOverscanRequested
2070                        || outsetsChanged) {
2071                    mLastSystemUiVisibility = mAttachInfo.mSystemUiVisibility;
2072                    mLastOverscanRequested = mAttachInfo.mOverscanRequested;
2073                    mAttachInfo.mOutsets.set(mPendingOutsets);
2074                    mApplyInsetsRequested = false;
2075                    dispatchApplyInsets(host);
2076                }
2077                if (visibleInsetsChanged) {
2078                    mAttachInfo.mVisibleInsets.set(mPendingVisibleInsets);
2079                    if (DEBUG_LAYOUT) Log.v(mTag, "Visible insets changing to: "
2080                            + mAttachInfo.mVisibleInsets);
2081                }
2082
2083                if (!hadSurface) {
2084                    if (mSurface.isValid()) {
2085                        // If we are creating a new surface, then we need to
2086                        // completely redraw it.  Also, when we get to the
2087                        // point of drawing it we will hold off and schedule
2088                        // a new traversal instead.  This is so we can tell the
2089                        // window manager about all of the windows being displayed
2090                        // before actually drawing them, so it can display then
2091                        // all at once.
2092                        newSurface = true;
2093                        mFullRedrawNeeded = true;
2094                        mPreviousTransparentRegion.setEmpty();
2095
2096                        // Only initialize up-front if transparent regions are not
2097                        // requested, otherwise defer to see if the entire window
2098                        // will be transparent
2099                        if (mAttachInfo.mThreadedRenderer != null) {
2100                            try {
2101                                hwInitialized = mAttachInfo.mThreadedRenderer.initialize(
2102                                        mSurface);
2103                                if (hwInitialized && (host.mPrivateFlags
2104                                        & View.PFLAG_REQUEST_TRANSPARENT_REGIONS) == 0) {
2105                                    // Don't pre-allocate if transparent regions
2106                                    // are requested as they may not be needed
2107                                    mSurface.allocateBuffers();
2108                                }
2109                            } catch (OutOfResourcesException e) {
2110                                handleOutOfResourcesException(e);
2111                                return;
2112                            }
2113                        }
2114                    }
2115                } else if (!mSurface.isValid()) {
2116                    // If the surface has been removed, then reset the scroll
2117                    // positions.
2118                    if (mLastScrolledFocus != null) {
2119                        mLastScrolledFocus.clear();
2120                    }
2121                    mScrollY = mCurScrollY = 0;
2122                    if (mView instanceof RootViewSurfaceTaker) {
2123                        ((RootViewSurfaceTaker) mView).onRootViewScrollYChanged(mCurScrollY);
2124                    }
2125                    if (mScroller != null) {
2126                        mScroller.abortAnimation();
2127                    }
2128                    // Our surface is gone
2129                    if (mAttachInfo.mThreadedRenderer != null &&
2130                            mAttachInfo.mThreadedRenderer.isEnabled()) {
2131                        mAttachInfo.mThreadedRenderer.destroy();
2132                    }
2133                } else if ((surfaceGenerationId != mSurface.getGenerationId()
2134                        || surfaceSizeChanged || windowRelayoutWasForced)
2135                        && mSurfaceHolder == null
2136                        && mAttachInfo.mThreadedRenderer != null) {
2137                    mFullRedrawNeeded = true;
2138                    try {
2139                        // Need to do updateSurface (which leads to CanvasContext::setSurface and
2140                        // re-create the EGLSurface) if either the Surface changed (as indicated by
2141                        // generation id), or WindowManager changed the surface size. The latter is
2142                        // because on some chips, changing the consumer side's BufferQueue size may
2143                        // not take effect immediately unless we create a new EGLSurface.
2144                        // Note that frame size change doesn't always imply surface size change (eg.
2145                        // drag resizing uses fullscreen surface), need to check surfaceSizeChanged
2146                        // flag from WindowManager.
2147                        mAttachInfo.mThreadedRenderer.updateSurface(mSurface);
2148                    } catch (OutOfResourcesException e) {
2149                        handleOutOfResourcesException(e);
2150                        return;
2151                    }
2152                }
2153
2154                final boolean freeformResizing = (relayoutResult
2155                        & WindowManagerGlobal.RELAYOUT_RES_DRAG_RESIZING_FREEFORM) != 0;
2156                final boolean dockedResizing = (relayoutResult
2157                        & WindowManagerGlobal.RELAYOUT_RES_DRAG_RESIZING_DOCKED) != 0;
2158                final boolean dragResizing = freeformResizing || dockedResizing;
2159                if (mDragResizing != dragResizing) {
2160                    if (dragResizing) {
2161                        mResizeMode = freeformResizing
2162                                ? RESIZE_MODE_FREEFORM
2163                                : RESIZE_MODE_DOCKED_DIVIDER;
2164                        // TODO: Need cutout?
2165                        startDragResizing(mPendingBackDropFrame,
2166                                mWinFrame.equals(mPendingBackDropFrame), mPendingVisibleInsets,
2167                                mPendingStableInsets, mResizeMode);
2168                    } else {
2169                        // We shouldn't come here, but if we come we should end the resize.
2170                        endDragResizing();
2171                    }
2172                }
2173                if (!mUseMTRenderer) {
2174                    if (dragResizing) {
2175                        mCanvasOffsetX = mWinFrame.left;
2176                        mCanvasOffsetY = mWinFrame.top;
2177                    } else {
2178                        mCanvasOffsetX = mCanvasOffsetY = 0;
2179                    }
2180                }
2181            } catch (RemoteException e) {
2182            }
2183
2184            if (DEBUG_ORIENTATION) Log.v(
2185                    TAG, "Relayout returned: frame=" + frame + ", surface=" + mSurface);
2186
2187            mAttachInfo.mWindowLeft = frame.left;
2188            mAttachInfo.mWindowTop = frame.top;
2189
2190            // !!FIXME!! This next section handles the case where we did not get the
2191            // window size we asked for. We should avoid this by getting a maximum size from
2192            // the window session beforehand.
2193            if (mWidth != frame.width() || mHeight != frame.height()) {
2194                mWidth = frame.width();
2195                mHeight = frame.height();
2196            }
2197
2198            if (mSurfaceHolder != null) {
2199                // The app owns the surface; tell it about what is going on.
2200                if (mSurface.isValid()) {
2201                    // XXX .copyFrom() doesn't work!
2202                    //mSurfaceHolder.mSurface.copyFrom(mSurface);
2203                    mSurfaceHolder.mSurface = mSurface;
2204                }
2205                mSurfaceHolder.setSurfaceFrameSize(mWidth, mHeight);
2206                mSurfaceHolder.mSurfaceLock.unlock();
2207                if (mSurface.isValid()) {
2208                    if (!hadSurface) {
2209                        mSurfaceHolder.ungetCallbacks();
2210
2211                        mIsCreating = true;
2212                        SurfaceHolder.Callback callbacks[] = mSurfaceHolder.getCallbacks();
2213                        if (callbacks != null) {
2214                            for (SurfaceHolder.Callback c : callbacks) {
2215                                c.surfaceCreated(mSurfaceHolder);
2216                            }
2217                        }
2218                        surfaceChanged = true;
2219                    }
2220                    if (surfaceChanged || surfaceGenerationId != mSurface.getGenerationId()) {
2221                        SurfaceHolder.Callback callbacks[] = mSurfaceHolder.getCallbacks();
2222                        if (callbacks != null) {
2223                            for (SurfaceHolder.Callback c : callbacks) {
2224                                c.surfaceChanged(mSurfaceHolder, lp.format,
2225                                        mWidth, mHeight);
2226                            }
2227                        }
2228                    }
2229                    mIsCreating = false;
2230                } else if (hadSurface) {
2231                    mSurfaceHolder.ungetCallbacks();
2232                    SurfaceHolder.Callback callbacks[] = mSurfaceHolder.getCallbacks();
2233                    if (callbacks != null) {
2234                        for (SurfaceHolder.Callback c : callbacks) {
2235                            c.surfaceDestroyed(mSurfaceHolder);
2236                        }
2237                    }
2238                    mSurfaceHolder.mSurfaceLock.lock();
2239                    try {
2240                        mSurfaceHolder.mSurface = new Surface();
2241                    } finally {
2242                        mSurfaceHolder.mSurfaceLock.unlock();
2243                    }
2244                }
2245            }
2246
2247            final ThreadedRenderer threadedRenderer = mAttachInfo.mThreadedRenderer;
2248            if (threadedRenderer != null && threadedRenderer.isEnabled()) {
2249                if (hwInitialized
2250                        || mWidth != threadedRenderer.getWidth()
2251                        || mHeight != threadedRenderer.getHeight()
2252                        || mNeedsRendererSetup) {
2253                    threadedRenderer.setup(mWidth, mHeight, mAttachInfo,
2254                            mWindowAttributes.surfaceInsets);
2255                    mNeedsRendererSetup = false;
2256                }
2257            }
2258
2259            if (!mStopped || mReportNextDraw) {
2260                boolean focusChangedDueToTouchMode = ensureTouchModeLocally(
2261                        (relayoutResult&WindowManagerGlobal.RELAYOUT_RES_IN_TOUCH_MODE) != 0);
2262                if (focusChangedDueToTouchMode || mWidth != host.getMeasuredWidth()
2263                        || mHeight != host.getMeasuredHeight() || contentInsetsChanged ||
2264                        updatedConfiguration) {
2265                    int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
2266                    int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);
2267
2268                    if (DEBUG_LAYOUT) Log.v(mTag, "Ooops, something changed!  mWidth="
2269                            + mWidth + " measuredWidth=" + host.getMeasuredWidth()
2270                            + " mHeight=" + mHeight
2271                            + " measuredHeight=" + host.getMeasuredHeight()
2272                            + " coveredInsetsChanged=" + contentInsetsChanged);
2273
2274                     // Ask host how big it wants to be
2275                    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
2276
2277                    // Implementation of weights from WindowManager.LayoutParams
2278                    // We just grow the dimensions as needed and re-measure if
2279                    // needs be
2280                    int width = host.getMeasuredWidth();
2281                    int height = host.getMeasuredHeight();
2282                    boolean measureAgain = false;
2283
2284                    if (lp.horizontalWeight > 0.0f) {
2285                        width += (int) ((mWidth - width) * lp.horizontalWeight);
2286                        childWidthMeasureSpec = MeasureSpec.makeMeasureSpec(width,
2287                                MeasureSpec.EXACTLY);
2288                        measureAgain = true;
2289                    }
2290                    if (lp.verticalWeight > 0.0f) {
2291                        height += (int) ((mHeight - height) * lp.verticalWeight);
2292                        childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(height,
2293                                MeasureSpec.EXACTLY);
2294                        measureAgain = true;
2295                    }
2296
2297                    if (measureAgain) {
2298                        if (DEBUG_LAYOUT) Log.v(mTag,
2299                                "And hey let's measure once more: width=" + width
2300                                + " height=" + height);
2301                        performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
2302                    }
2303
2304                    layoutRequested = true;
2305                }
2306            }
2307        } else {
2308            // Not the first pass and no window/insets/visibility change but the window
2309            // may have moved and we need check that and if so to update the left and right
2310            // in the attach info. We translate only the window frame since on window move
2311            // the window manager tells us only for the new frame but the insets are the
2312            // same and we do not want to translate them more than once.
2313            maybeHandleWindowMove(frame);
2314        }
2315
2316        final boolean didLayout = layoutRequested && (!mStopped || mReportNextDraw);
2317        boolean triggerGlobalLayoutListener = didLayout
2318                || mAttachInfo.mRecomputeGlobalAttributes;
2319        if (didLayout) {
2320            performLayout(lp, mWidth, mHeight);
2321
2322            // By this point all views have been sized and positioned
2323            // We can compute the transparent area
2324
2325            if ((host.mPrivateFlags & View.PFLAG_REQUEST_TRANSPARENT_REGIONS) != 0) {
2326                // start out transparent
2327                // TODO: AVOID THAT CALL BY CACHING THE RESULT?
2328                host.getLocationInWindow(mTmpLocation);
2329                mTransparentRegion.set(mTmpLocation[0], mTmpLocation[1],
2330                        mTmpLocation[0] + host.mRight - host.mLeft,
2331                        mTmpLocation[1] + host.mBottom - host.mTop);
2332
2333                host.gatherTransparentRegion(mTransparentRegion);
2334                if (mTranslator != null) {
2335                    mTranslator.translateRegionInWindowToScreen(mTransparentRegion);
2336                }
2337
2338                if (!mTransparentRegion.equals(mPreviousTransparentRegion)) {
2339                    mPreviousTransparentRegion.set(mTransparentRegion);
2340                    mFullRedrawNeeded = true;
2341                    // reconfigure window manager
2342                    try {
2343                        mWindowSession.setTransparentRegion(mWindow, mTransparentRegion);
2344                    } catch (RemoteException e) {
2345                    }
2346                }
2347            }
2348
2349            if (DBG) {
2350                System.out.println("======================================");
2351                System.out.println("performTraversals -- after setFrame");
2352                host.debug();
2353            }
2354        }
2355
2356        if (triggerGlobalLayoutListener) {
2357            mAttachInfo.mRecomputeGlobalAttributes = false;
2358            mAttachInfo.mTreeObserver.dispatchOnGlobalLayout();
2359        }
2360
2361        if (computesInternalInsets) {
2362            // Clear the original insets.
2363            final ViewTreeObserver.InternalInsetsInfo insets = mAttachInfo.mGivenInternalInsets;
2364            insets.reset();
2365
2366            // Compute new insets in place.
2367            mAttachInfo.mTreeObserver.dispatchOnComputeInternalInsets(insets);
2368            mAttachInfo.mHasNonEmptyGivenInternalInsets = !insets.isEmpty();
2369
2370            // Tell the window manager.
2371            if (insetsPending || !mLastGivenInsets.equals(insets)) {
2372                mLastGivenInsets.set(insets);
2373
2374                // Translate insets to screen coordinates if needed.
2375                final Rect contentInsets;
2376                final Rect visibleInsets;
2377                final Region touchableRegion;
2378                if (mTranslator != null) {
2379                    contentInsets = mTranslator.getTranslatedContentInsets(insets.contentInsets);
2380                    visibleInsets = mTranslator.getTranslatedVisibleInsets(insets.visibleInsets);
2381                    touchableRegion = mTranslator.getTranslatedTouchableArea(insets.touchableRegion);
2382                } else {
2383                    contentInsets = insets.contentInsets;
2384                    visibleInsets = insets.visibleInsets;
2385                    touchableRegion = insets.touchableRegion;
2386                }
2387
2388                try {
2389                    mWindowSession.setInsets(mWindow, insets.mTouchableInsets,
2390                            contentInsets, visibleInsets, touchableRegion);
2391                } catch (RemoteException e) {
2392                }
2393            }
2394        }
2395
2396        if (mFirst) {
2397            if (sAlwaysAssignFocus || !isInTouchMode()) {
2398                // handle first focus request
2399                if (DEBUG_INPUT_RESIZE) {
2400                    Log.v(mTag, "First: mView.hasFocus()=" + mView.hasFocus());
2401                }
2402                if (mView != null) {
2403                    if (!mView.hasFocus()) {
2404                        mView.restoreDefaultFocus();
2405                        if (DEBUG_INPUT_RESIZE) {
2406                            Log.v(mTag, "First: requested focused view=" + mView.findFocus());
2407                        }
2408                    } else {
2409                        if (DEBUG_INPUT_RESIZE) {
2410                            Log.v(mTag, "First: existing focused view=" + mView.findFocus());
2411                        }
2412                    }
2413                }
2414            } else {
2415                // Some views (like ScrollView) won't hand focus to descendants that aren't within
2416                // their viewport. Before layout, there's a good change these views are size 0
2417                // which means no children can get focus. After layout, this view now has size, but
2418                // is not guaranteed to hand-off focus to a focusable child (specifically, the edge-
2419                // case where the child has a size prior to layout and thus won't trigger
2420                // focusableViewAvailable).
2421                View focused = mView.findFocus();
2422                if (focused instanceof ViewGroup
2423                        && ((ViewGroup) focused).getDescendantFocusability()
2424                                == ViewGroup.FOCUS_AFTER_DESCENDANTS) {
2425                    focused.restoreDefaultFocus();
2426                }
2427            }
2428        }
2429
2430        final boolean changedVisibility = (viewVisibilityChanged || mFirst) && isViewVisible;
2431        final boolean hasWindowFocus = mAttachInfo.mHasWindowFocus && isViewVisible;
2432        final boolean regainedFocus = hasWindowFocus && mLostWindowFocus;
2433        if (regainedFocus) {
2434            mLostWindowFocus = false;
2435        } else if (!hasWindowFocus && mHadWindowFocus) {
2436            mLostWindowFocus = true;
2437        }
2438
2439        if (changedVisibility || regainedFocus) {
2440            // Toasts are presented as notifications - don't present them as windows as well
2441            boolean isToast = (mWindowAttributes == null) ? false
2442                    : (mWindowAttributes.type == WindowManager.LayoutParams.TYPE_TOAST);
2443            if (!isToast) {
2444                host.sendAccessibilityEvent(AccessibilityEvent.TYPE_WINDOW_STATE_CHANGED);
2445            }
2446        }
2447
2448        mFirst = false;
2449        mWillDrawSoon = false;
2450        mNewSurfaceNeeded = false;
2451        mActivityRelaunched = false;
2452        mViewVisibility = viewVisibility;
2453        mHadWindowFocus = hasWindowFocus;
2454
2455        if (hasWindowFocus && !isInLocalFocusMode()) {
2456            final boolean imTarget = WindowManager.LayoutParams
2457                    .mayUseInputMethod(mWindowAttributes.flags);
2458            if (imTarget != mLastWasImTarget) {
2459                mLastWasImTarget = imTarget;
2460                InputMethodManager imm = InputMethodManager.peekInstance();
2461                if (imm != null && imTarget) {
2462                    imm.onPreWindowFocus(mView, hasWindowFocus);
2463                    imm.onPostWindowFocus(mView, mView.findFocus(),
2464                            mWindowAttributes.softInputMode,
2465                            !mHasHadWindowFocus, mWindowAttributes.flags);
2466                }
2467            }
2468        }
2469
2470        // Remember if we must report the next draw.
2471        if ((relayoutResult & WindowManagerGlobal.RELAYOUT_RES_FIRST_TIME) != 0) {
2472            reportNextDraw();
2473        }
2474
2475        boolean cancelDraw = mAttachInfo.mTreeObserver.dispatchOnPreDraw() || !isViewVisible;
2476
2477        if (!cancelDraw && !newSurface) {
2478            if (mPendingTransitions != null && mPendingTransitions.size() > 0) {
2479                for (int i = 0; i < mPendingTransitions.size(); ++i) {
2480                    mPendingTransitions.get(i).startChangingAnimations();
2481                }
2482                mPendingTransitions.clear();
2483            }
2484
2485            performDraw();
2486        } else {
2487            if (isViewVisible) {
2488                // Try again
2489                scheduleTraversals();
2490            } else if (mPendingTransitions != null && mPendingTransitions.size() > 0) {
2491                for (int i = 0; i < mPendingTransitions.size(); ++i) {
2492                    mPendingTransitions.get(i).endChangingAnimations();
2493                }
2494                mPendingTransitions.clear();
2495            }
2496        }
2497
2498        mIsInTraversal = false;
2499    }
```

内容比较多，可以确定顺序是先测量再布局最后绘制。



# performDraw的顺序



# 关于DisplayList（显示列表）

View的draw源码里有很多关于DisplayList的代码，这里介绍一下关于DisplayList的设计。

启用硬件加速后，Android 框架会采用新的绘制模型，该模型利用显示列表将应用渲染到屏幕上。

Android 系统仍会使用 `invalidate()` 和 `draw()` 请求屏幕更新和渲染视图，但会采用其他方式处理实际绘制过程。Android 系统不会立即执行绘制命令，而是将这些命令记录在显示列表中，这些列表中包含视图层次结构绘制代码的输出。另一项优化是，Android 系统只需要记录和更新被 `invalidate()` 调用标记为脏视图的视图的显示列表。只需重新发出之前记录的显示列表，即可重新绘制未经过无效化处理的视图。新绘制模型包含以下三个阶段：

1. 对层次结构进行无效化处理
2. 记录并更新显示列表
3. 绘制显示列表

使用此模型时，您无法依赖与脏区域交互的视图来执行其 `draw()` 方法。要确保 Android 系统会记录视图的显示列表，您必须调用 `invalidate()`。如果忘记执行此操作，则视图在发生更改后看起来仍然没有变化。

使用显示列表还有助于改进动画性能，因为设置特定属性（例如 Alpha 或旋转）不需要对目标视图进行无效化处理（该操作是自动完成的）。这项优化还适用于具有显示列表的视图（如果应用经过硬件加速，则适用于所有视图）。例如，假设有一个 `LinearLayout`，其中包含一个`ListView`（位于 `Button` 之上）。`LinearLayout` 的显示列表如下所示：

- DrawDisplayList(ListView)
- DrawDisplayList(Button)

假设您现在要更改 `ListView` 的不透明度。在对 `ListView` 调用 `setAlpha(0.5f)` 后，显示列表现在包含以下内容：

- SaveLayerAlpha(0.5)
- DrawDisplayList(ListView)
- Restore
- DrawDisplayList(Button)

系统没有执行 `ListView` 的复杂绘制代码，而是仅更新了更为简单的 `LinearLayout` 的显示列表。在未启用硬件加速的应用中，系统会再次执行列表及其父级的绘制代码。



