# **The Fundamentals of User Interfaces - 0.98a**

The goal of this resource is to teach the **Graphical User Interface** system implemented by Starsector and how to work with it. This resource does not cover UI design in general or how to achieve a good UX.

<br><br>

# **The GUI System**

<br>

## **The UI Tree**

Starsector has 4 `AppState`'s in total: `TitleScreenState`, `CampaignState`, `CombatState` and `GLLauncher`. The state `GLLauncher` is not relevant for this resource, thus it will be ignored.

Each `AppState` has a UI Tree. But in general, each `UIPanelAPI` implementation can be thought of as a tree. It would be more accurate to state, that each `AppState` has its own root panel.

Do keep in mind the actual signature of the class implementing `UIPanelAPI` is obfuscated, and thus cannot be referenced directly, but for the purposes of this resource, it will be referred to as `UIPanel`. Same for `UIComponent`, which refers to the obfuscated class implementing `UIComponentAPI`, and `Position`, which refers to the obfuscated `PositionAPI` implementation.

<details>
<summary>Code Snippet</summary>

```java
public class TitleScreenState extends BaseGameState implements ... {
    private UIPanel screenPanel;

    @Override
    public UIPanel getScreenPanel() {
        return screenPanel;
    }
}

public class CampaignState extends BaseGameState implements ... {
    private UIPanel screenPanel;

    public UIPanel getScreenPanel() {
        return screenPanel;
    }

    @Override
    public UIPanel getDialogParent() {
        return screenPanel;
    }
}

public class CombatState implements AppState, ... {
    private UIPanel widgetPanel;

    public UIPanel getWidgetPanel() {
        return widgetPanel;
    }

    @Override
    public UIPanel getScreenPanel() {
        return widgetPanel;
    }
}
```
</details>

Each `AppState`, with the exception of `GLLauncher`, also has a `codexOverlay` panel. It's lifecycle differs from the `screenPanel` in general, but it can also be accessed using the `UIPanel getOverlayPanelForCodex()` method, which all three `AppState`'s possess. The panel of focus will be the `screenPanel`. Further, please pretend the field name for the root panel that `CombatState` has is `screenPanel` and not `widgetPanel`.

Unlike branch panels or leaf panels inside the UI Tree/Hierarchy, the root panel is special. It does not have a `Position` parent, thus its rendering must be absolute, despite `Position` being designed for relative positioning. It does not have a parent to dictate its traversal order, thus the `AppState` must manage the lifecycle manually. Let us see how each `AppState` manages the lifecycle of their `screenPanel`.

### **TitleScreenState Root Panel**
When the GUI of `TitleScreenState` is created, `screenPanel` is positioned at <0, 0> with the dimensions <screen_width, screen_height>. Then two panels, `titleWidget` and `missionWidget` are added to it (as a child).
<details>
<summary>Code snippet</summary>

```java
screenPanel = new UIPanel(new Position(0f, 0f, Global.getSettings().getScreenWidth(), Global.getSettings().getScreenHeight()));
titleWidget = new TitleWidget(this);
titleWidget.getPosition().inMid();
screenPanel.add(titleWidget);
missionWidget = new MissionWidget(this);
screenPanel.add(missionWidget);
```
</details>

The traversal order for `TitleScreenState` is inherited from `BaseGameState`, which first does the `render` pass, then the `processInput` pass, and lastly the `advance` pass. During the render pass, the initial alpha value passed down to `screenPanel` as a parameter is 1. 

### **CampaignState Root Panel**
When the GUI of `CampaignState` is created, `screenPanel` is positioned at <0, 0> with the dimensions <screen_width, screen_height>. Many panels are added to the root panel. I will not list them all.
<details>
<summary>Code snippet</summary>

```java
screenPanel = new UIPanel(new Position(0f, 0f, Global.getSettings().getScreenWidth(), Global.getSettings().getScreenHeight()));
codexOverlay = new UIPanel(new Position(0f, 0f, Global.getSettings().getScreenWidth(), Global.getSettings().getScreenHeight()));
...
core = new CoreUI(this, null, false, screenPanel);
screenPanel.add(core).inBL(0f, 0f);
messageListV3 = new MessageListV3();
screenPanel.add(messageListV3).setSize(1024f, 400f).inBL(10f, 287f);
screenPanel.add(messageDisplay).setSize(Global.getSettings().getScreenWidth(), 20f).inTMid(5.0f);
belowTooltips = new z(new Position(0f, 0f, Global.getSettings().getScreenWidth(), Global.getSettings().getScreenHeight())){

    @Override
    protected void renderImpl(float alpha) {
        super.renderImpl(alpha);
        CampaignState.this.renderInUICoordsAboveUIBelowTooltips();
    }
};
screenPanel.add(belowTooltips).inTL(0f, 0f);
screenPanel.pack(); /** recomputes the Position's of all of its children recursively */
screenPanel.bringToTop(belowTooltips);
```
</details>

The traversal order for `CampaignState` is inherited from `BaseGameState`. The render alpha value is also 1.

### **CombatState Root Panel**
When the GUI of `CombatState` is created, `screenPanel` is positioned at <0, 0> with the dimensions <screen_width, screen_height>. The panels `ribbon` and `shipInfo` are added to it.
<details>
<summary>Code snippet</summary>

```java
widgetPanel = new UIPanel(new Position(0f, 0f, Global.getSettings().getScreenWidth(), Global.getSettings().getScreenHeight()));
codexOverlay = new UIPanel(new Position(0f, 0f, Global.getSettings().getScreenWidth(), Global.getSettings().getScreenHeight()));
showingCodex = false;
ribbon = new Ribbon(Global.getSettings().getScreenWidth(), Global.getSettings().getScreenHeight());
targetReticleRenderer = new TargetReticleRenderer();
messageWidget = new MessageWidget();
widgetPanel.add(ribbon).inBL(0f, 0f);
shipInfo = new ShipInfo();
widgetPanel.add(shipInfo).setSize(800f, 200f).inBL(20f, 20f);
```
</details>

The `traverse` implementation of `CombatState` differ from BaseGameState, mainly to handle the custom logic needed for `CombatViewport`, but the traverse order for the root panel is identical to `TitleScreenState` and `CampaignState`:
`render -> processInput -> advance`

<br><br>

## **UIComponentAPI**

Having covered the lifecycle of the `screenPanel` for all `AppState`'s, the next piece of the puzzle in understanding how the GUI works is to learn how the most elemental building block of the UI Tree works. `UIComponent` in Starsector is bit of a bundle, as it handles many features that it, in my opinion, should not handle at all. Not every aspect of `UIComponent` will be covered, as there are too many. Before we begin, here is a member list of `UIComponent`, renamed for human readability and semantic correctness alongside comments by me.

<details>
<summary>Code snippet</summary>

```java
/** Used for layout/position calculations during traversal. */
private Position mPos = new Position(this);
/** Tooltips are attached to the UIComponent, which manages the lifecycle of its tooltip. Type is an obfuscated interface that all TooltipMakerAPI instances implement, so don't worry about it. */
private Oo0o mTooltip;
/** Handles processInput for the tooltip. Positioning as well (if I understood it correctly). Don't mind the obfuscated type. */
private classsuper mTooltipLogic;
/** The main fader used when making a panel invisible with transition or vice versa. */
private Fader mFader = new Fader(1f, 0.05f, 0.25f);
/** Used to slide elements sliding inside and outside the screen. Think info bar widget, ability bar, mini-map on the right side etc. */
private Fader mSlideFader = new Fader(1f, 0.25f, 0.25f);
/** Fades in when hovered, fades out when not. Used for hover glow and similar. */
private Fader mMouseoverHighlightFader = new Fader(0f, 0.1f, 0.1f);
/** Used by mSlideFader, where the position offsets take full effect when mSlideFader is dark and have no effect when mSlideFader is bright. */
private float sliderOffsetX;
/** Same as sliderOffsetX but for the Y axis. */
private float sliderOffsetY;
/** Not used internally. Mostly used by the button panel, but button has its own isEnabled function, so idk what this is doing here. */
protected boolean enabled = true;
/** The logical parent of this UIComponent. Allows the panel to access its parent. */
private UIPanel mParent;
/** A wrapper class for listeners. Used only in 2 places by the codebase. Genuinely no idea why this was placed here. Not used internally. */
private ObjectRepository listeners = new ObjectRepository();
/** Base opacity of the UIComponent used for rendering. */
private float mOpacity = 1f;
/** Used internally to guard against constant offset updates even when the panel is not sliding. Recalculating positions is expensive, so this is significant. */
private boolean mFinishedSlidingIn = true;
/** Same as mFinishedSlidingIn. */
private boolean mFinishedSlidingOut = true;
/** The amount of ticks for which the UIComponent was not sliding in/out of the screen. Only used for the info bar widget, the ability bar and the mini-map. */
private int slideIdleTickCount = 0;
/** Affects the hover hitbox of the UIComponent, used only by mMouseoverHighlightFader. */
private float mMousePadXLeft = 0f;
/** Same as mMousePadXLeft. */
private float mMousePadXRight = 0f;
/** Same as mMousePadXLeft. */
private float mMousePadYTop = 0f;
/** Same as mMousePadXLeft. */
private float mMousePadYBottom = 0f;
/** Used internally to manage tooltip state. */
private boolean mTooltipVisible = false;
/** If present, offsets the tooltip owned by this UIComponent from its center by the provided values. */
private Vector2f mTooltipCenterOffset = null;
/** If a mTooltipAnchor is present, adds these values to the coordinates of the anchor to position the tooltip. */
private Vector2f mTooltipAnchorOffset = null;
/** The panel the tooltip should be anchored to if present. The type is an interface (that implements UIComponentAPI) with some convenience methods like getX, getY etc. */
private OO0o mTooltipAnchor = null;
/** If present, its tooltip reset logic will be used instead of mTooltipLogic's tooltip reset logic. */
private classsuper mTooltipLogicResetDelegate = null;
/** 
    * If passed down to showTooltip() as a parameter, the method skips calling beforeShown() on the tooltip, does not fade the tooltip in and does not
    * detach the tooltip should the parent of UIComponent be a scroll panel and currently being scrolled (tooltips disappear in vanilla when scrolling).
    */
public static java.lang.Object SOURCE_UPDATE_POSITION_ONLY = new java.lang.Object();
/** What it says on the tin. */
private boolean mForceTooltipToLowerRightOfMouse = false;
/** Used to call afterSizeFirstChanged, which is used in 9 places. Needed by vanilla, because Alex likes to set panel size after creating it. */
protected boolean created = false;
```
</details>

The first things that piques interest are the traversal methods `render`, `processInput` and `advance`, which are already implemented by `UIComponent`, and instead call the abstract methods `renderImpl`, `processInputImpl` and `advanceImpl` for child classes to override. This common pattern is used to lock in certain logic that almost all panels have to abide by. However, the primary traversal methods are not final, so they can also be overridden if needs be. Let us see what each traversal method does:

### **render**
Perhaps the simplest, this method calculates brightness values using `mSlideFader`, `mFader` and `mOpacity`. If the value is less than or equal to 0, it skips the main render method, otherwise it calls `renderImpl`.
<details>
<summary>Code snippet</summary>

```java
public void render(float alpha) {
    float slideBrightness = Math.min(mSlideFader.getBrightness() * 2.0f, 1.0f);
    alpha *= mOpacity * slideBrightness;
    if (alpha <= 0.0f) return;
    
    this.renderImpl(alpha * mFader.getBrightness());
}
```
</details>

### **processInput**
If the `UIComponent` is invisible or slid outside the screen, then no events are processed. Otherwise it is determined whether if the tooltip should be shown by looking at the active status of the component. If certain conditions pass, the tooltip gets to handle the events first. Then if the panel is active, and no other element has focus or if the focused element is the panel itself, its `mMouseoverHighlightFader` is updated. Lastly `processInputImpl` is called.
<details>
<summary>Code snippet</summary>

```java
public void processInput(List<InputEventAPI> list) {
    if (mSlideFader.isFadedOut() || getFader().isFadedOut() || getOpacity() <= 0.0f) {
        return;
    }
    boolean isActive = true;
    boolean showTpWhenInactive = false;
    if (this instanceof Button) {
        n n2 = (n)this;
        isActive = n2.isActive();
        showTpWhenInactive = n2.isShowTooltipWhileInactive();
    }
    if ((isActive || showTpWhenInactive) && mTooltip != null && mTooltipLogic != O0Oo.\u00d300000() && mTooltipLogic != null) {
        mTooltipLogic.processInput(list);
    }
    if (!isActive) return;

    boolean isHovering = false;
    /** If no element is focused or the focused element is the tooltip logic manager. */
    if (O0Oo.\u00d300000() == null || O0Oo.\u00d300000() == mTooltipLogic) {
        for (InputEventAPI event : list) {
            if (event.isConsumed() || !event.isMouseMoveEvent() || !getPosition().containsLocation(event.getX(), event.getY(), mMousePadXLeft, mMousePadXRight, mMousePadYTop, mMousePadYBottom)) continue;
            isHovering = true;
            break;
        }
    }
    if (isHovering) {
        mMouseoverHighlightFader.fadeIn();
    } else {
        mMouseoverHighlightFader.fadeOut();
    }
    processInputImpl(list);
}
```
</details>

### **advance**
The method advances the faders, handles sliding logic, advances `mTooltipLogic`, calls advanceImpl and detaches the tooltip from its parent (often `screenPanel`) if it is completely faded out. Not much to say ngl.
<details>
<summary>Code snippet</summary>

```java
public void advance(float delta) {
    mFader.advance(delta * Global.getSettings().getFloat("uiFadeSpeedMult"));
    mSlideFader.advance(delta * Global.getSettings().getFloat("uiFadeSpeedMult"));
    mMouseoverHighlightFader.advance(delta);
    if (!mFinishedSlidingIn || !mFinishedSlidingOut) {
        updateSlidingOffsets();
        if (mSlideFader.getBrightness() == 1.0f) {
            mFinishedSlidingIn = true;
        }
        if (mSlideFader.getBrightness() == 0.0f) {
            mFinishedSlidingOut = true;
        }
    }
    slideIdleTickCount = mSlideFader.isIdle() ? ++slideIdleTickCount : 0;
    if (mTooltipLogic != null) {
        mTooltipLogic.advance(delta);
    }
    advanceImpl(delta);

    if (mTooltipVisible && mTooltip.getFader().isFadedOut()) {
        UIPanel tooltipParent = findTopAncestor();
        if (tooltipParent != null) {
            tooltipParent.remove(mTooltip);
        }
        mTooltipVisible = false;
        mTooltip.notiftyHidden();
    }
}
```
</details>

### **tooltip**
The details of how the tooltip lifecycle is managed is irrelevant, but the fact that `UIComponent` manages it gives us a hint as to their usage. The following methods from `StandardTooltipV2Expandable` are what vanilla uses to attach to `UIComponent` its tooltip in most cases. Using this method with reflection on any vanilla panel is a valid approach. How to use StandardTooltipV2 will be covered later.

```java
public static void addTooltipBelow(final UIComponent comp, final StandardTooltipV2 standardTooltipV2);
public static void addTooltipBelowRight(final UIComponent comp, final StandardTooltipV2 standardTooltipV2);
public static void addTooltipLeft(final UIComponent comp, final StandardTooltipV2 standardTooltipV2);
public static void addTooltipLeft(final UIComponent comp, final StandardTooltipV2 standardTooltipV2, float gap);
public static void addTooltipRight(final UIComponent comp, final StandardTooltipV2 standardTooltipV2);
public static void addTooltipAbove(final UIComponent comp, final StandardTooltipV2 standardTooltipV2);
```

To summarize, the abstract `UIComponent` is the elemental UI class that is responsible for handling positioning, faders, events, basic rendering and its own tooltip. There is no API method to instance it, as it is abstract.

<br><br>

## **UIPanelAPI**

Directly extending `UIComponent`, the `UIPanelAPI` adds only two new things: a list of children and the scissor stack. The children are stored inside a list of internal `UIComponentAPI`'s. Internal vanilla code uses an extended `UIComponentAPI` to store its children, which I will call `InternalUIComponentAPI`. The scissor stack, which is a static utility class, lets the UI Tree have multiple scissor bounds without forgetting them. Cases where multiple scissors are used is indeed rare, but it can be useful nonetheless. Further, the panel maintains a copy list with identical children that get updated from time to time. I would not have done it that way, but to each their own. Methods of relevance are how children are added and removed, how children are sent to bottom and brought to top, and how the traversal methods are modified to handle the children.

<details>
<summary>Code snippet</summary>

```java
private List<InternalUIComponentAPI> children = new ArrayList<>();
protected boolean needToCopy = true;
protected List<InternalUIComponentAPI> copy = new ArrayList<>();
private boolean doClipping = false;
```
</details>

When adding a child to a `UIPanel`, the child itself is not only added to the children list, but its position is also added to the parent position's children list. Then, the parent of the child is set to the parent alongside the child position's parent position.

<details>
<summary>Code snippet</summary>

```java
public Position add(InternalUIComponentAPI comp) {
    this.needToCopy = true;
    this.children.add(comp);
    comp.getPosition().setParent(this.getPosition());
    this.getPosition().add(comp.getPosition());
    comp.setParent(this);
    return comp.getPosition();
}

public PositionAPI addComponent(UIComponentAPI comp) {
    return this.add((InternalUIComponentAPI) comp);
}
```
</details>

When a child is submitted for removal, it is first checked whether if the child has a tooltip and removed if it does. Then, the child is removed from the child list and it's position is removed from the parent position's child list. The position parent and the parent of the child are set to null.

<details>
<summary>Code snippet</summary>

```java
public void remove(InternalUIComponentAPI comp) {
    if (comp == null) return;

    if (comp instanceof UIComponent uiComp && uiComp.getTooltip() != null) {
        uiComp.forceRemoveTooltip();
    }
    this.needToCopy = true;
    this.children.remove(comp);
    comp.getPosition().setParent(null);
    this.getPosition().remove(comp.getPosition());
    comp.setParent(null);
}

public void removeComponent(UIComponentAPI comp) {
    this.remove((InternalUIComponentAPI) comp);
}
```
</details>

When bringing a child to the visual top (rendered last), child is removed from the child list and placed at the end. Then, the parent calls `bringToTop` on its own parent if present with itself as the parameter. This ensures the entire tree branch is rendered last. The method `bringToTopWithinItself` works the same way, except the parent does not call its own parent's `bringToTop` method.

<details>
<summary>Code snippet</summary>

```java
public void bringToTop(InternalUIComponentAPI comp) {
    if (comp == null) return;
    
    boolean isAlreadyLast = !children.isEmpty() && children.get(children.size() - 1) == comp;
    if (!isAlreadyLast) {
        remove(comp);
        add(comp);
    }
    if (getParent() != null) {
        getParent().bringToTop(this);
    }
}

public void bringComponentToTop(UIComponentAPI comp) {
    this.bringToTop((InternalUIComponentAPI) comp);
}
```
</details> 

When bringing a child to the visual bottom (rendered first), child is removed from the child list and placed at the beginning. Then, the parent calls `sendToBottom` on its own parent if present with itself as the parameter. This ensures the entire tree branch is rendered first. The method `sendToBottomWithinItself` works the same way, except the parent does not call its own parent's `sendToBottom` method. Surprisingly, the parameter type for `sendToBottom` is `UIComponentAPI` and not `InternalUIComponentAPI`. Also, the position is also re-added at the beginning, just in case I guess.

<details>
<summary>Code snippet</summary>

```java
public void sendToBottom(UIComponentAPI comp) {
    if (comp == null) return;
    
    boolean bl = !children.isEmpty() && children.get(0) == comp;
    if (!bl) {
        remove((OO0o)comp);
        needToCopy = true;
        children.add(0, (OO0o)comp);
        ((InternalUIComponentAPI)comp).getPosition().setParent(getPosition());
        getPosition().add(0, comp.getPosition());
        ((InternalUIComponentAPI) comp).setParent(this);
    }
    if (getParent() != null) {
        getParent().sendToBottom(this);
    }
}
```
</details>

The `render` method itself is not overridden. Instead the `renderImpl` method is modified to enable scissoring if requested, render all children sequentially and pop the scissor stack afterwards. Please note that the actual scissor stack class name is obfuscated, but referred to as `ScissorStack` in this example.

<details>
<summary>Code snippet</summary>

```java
@Override
protected void renderImpl(float alpha) {
    if (alpha <= 0.0f) return;
    
    updateCopyIfNeeded();
    if (isClipping()) {
        Position object = getPosition();
        float x = object.getX();
        float y = object.getY();
        float w = object.getWidth();
        float h = object.getHeight();
        ScissorStack.push((int)x, (int)y, (int)w, (int)h);
    }
    for (InternalUIComponentAPI child : copy) {
        child.render(alpha);
    }
    if (isClipping()) {
        ScissorStack.pop();
    }
}
```
</details>

The `processInput` method is also not overridden. `processInputImpl` calls `processInput` on its children in reverse order, such that panels visually on the top get to handle the events first. This is consistent with what the User would expect.

<details>
<summary>Code snippet</summary>

```java
@Override
protected void processInputImpl(List<InputEventAPI> list) {
    if (getFader().isFadingOut() || getFader().getBrightness() == 0f) {
        return;
    }
    dispatchEventsToChildren(list);
}

protected void dispatchEventsToChildren(List<InputEventAPI> list) {
    updateCopyIfNeeded();
    List<InternalUIComponentAPI> childrenCopy = copy;
    int index = childrenCopy.size() - 1;
    while (index >= 0) {
        childrenCopy.get(index).processInput(list);
        --index;
    }
}
```
</details>

Lastly, `advanceImpl` updates the child copy list and calls `advance` on its children sequentially.

<details>
<summary>Code snippet</summary>

```java
@Override
protected void advanceImpl(float delta) {
    updateCopyIfNeeded();
    for (InternalUIComponentAPI child : copy) {
        child.advance(delta);
    }
}
```
</details>

There also is no way to get a raw instance of a `UIPanel` through the vanilla API.

<br><br>

## **CustomPanelAPI**

The last "elemental" panel is the `CustomPanelAPI` implementation, which I will call `CustomPanel`. It extends `UIPanel` and stores a plugin `mPlugin`, to which it forwards traversal calls. Further, it has methods for creating normal and scrollable tooltips. It also holds an `IntelUIAPI` and `EventsPanel` instance for easy use within the Intel tab. I would consider the current implementation of `CustomPanelAPI` to be cramped, as it contains methods for creating a tooltip, which should really be separate from the panel itself. The tooltip capabilities and plugin integration of `CustomPanel` will be covered.

<details>
<summary>Code snippet</summary>

```java
/** The plugin to which the panel forwards the traversal calls. */
private CustomUIPanelPlugin mPlugin;
/** The initial width of the panel upon creation is stored here. Not used anywhere. Has no getters or setters. */
private float mInitialWidth;
/** The initial height of the panel upon creation is stored here. Not used anywhere. Has no getters or setters. */
private float mInitialHeight;
/** In the context of an IntelUI panel, this can be accessed using a getter. */
private EventsPanel mEventPanel;
/** In the context of an IntelUI panel, this can be accessed using a getter. */
private IntelUIAPI mIntelUI;
/** When using a scrollable tooltip this acts as the view panel height. When used for standard tooltips, acts as the minimum height of the tooltip. */
private Map<StandardTooltipV2Expandable, Float> mTooltipToHeight = new HashMap<>();
/** Remembers the value of the withScroller parameter when the createUIElement method is called. */
private Map<StandardTooltipV2Expandable, Boolean> mTooltipToHasScrollbar = new HashMap<>();
```
</details>

Nothing special is done for `renderImpl`, `processInputImpl` and `advanceImpl`. For all three methods, first the super method is called, then the `mPlugin` equivalent is called. Except for `renderImpl`, the `renderBelow` method of `mPlugin` is called first. The `positionChanged` method of the plugin also gets called whenever the panel moves or changes size.

<details>
<summary>Code snippet</summary>

```java
@Override
protected void renderImpl(float alpha) {
    if (mPlugin != null) {
        mPlugin.renderBelow(alpha);
    }
    super.renderImpl(alpha);
    if (mPlugin != null) {
        mPlugin.render(alpha);
    }
}

@Override
protected void advanceImpl(float delta) {
    super.advanceImpl(delta);
    if (mPlugin != null) {
        mPlugin.advance(delta);
    }
}

@Override
protected void processInputImpl(List<InputEventAPI> inputs) {
    super.processInputImpl(inputs);
    if (mPlugin != null) {
        /** Events are copied for the plugin for some reason. */
        mPlugin.processInput(new ArrayList<>(inputs));
    }
}
```
</details>

Other methods of `CustomPanel` are related to tooltip creation. The method `createUIElement` provides a tooltip with no border or background. It also removes itself when it fades out. This tooltip is useful only for UI layouting and other functionality like tables, grids etc. provided by the tooltip. It cannot be used to create an actual tooltip. This will be covered later. The height of the tooltip and its scroll bar status are stored by `mTooltipToHeight` and `mTooltipToHasScrollbar`. Additionally, should the tooltip have a scrollbar, 5 pixels of width are reserved for the scrollbar.

<details>
<summary>Code snippet</summary>

```java
public TooltipMakerAPI createUIElement(float width, float height, boolean withScroller) {
    StandardTooltipV2Expandable standardTooltipV2Expandable = StandardTooltipV2Expandable.createAsUIElement(width - (withScroller ? 5.0f : 0.0f));
    if (mEventPanel != null) {
        standardTooltipV2Expandable.setButtonListener(mEventPanel);
    } else {
        standardTooltipV2Expandable.setButtonListener(this);
    }
    standardTooltipV2Expandable.setForceProcessInput(true);
    mTooltipToHeight.put(standardTooltipV2Expandable, Float.valueOf(height));
    mTooltipToHasScrollbar.put(standardTooltipV2Expandable, withScroller);
    return standardTooltipV2Expandable;
}
```
</details>

When it comes time to attach the tooltip to the panel it was created with, and do note that for it to work properly it needs to be attached to the `CustomPanel` that created it, the `addUIElement` method is used. This handy wrapper calls certain methods to update the dimensions of the tooltip and wraps it around a scroll panel if during the creation of the tooltip this was requested. The `ScrollPanelAPI` implementation is obfuscated, but I will refer to it as `ScrollPanel`.

<details>
<summary>Code snippet</summary>

```java
public PositionAPI addUIElement(TooltipMakerAPI tooltip) {
    StandardTooltipV2Expandable tp = (StandardTooltipV2Expandable)tooltip;
    StandardTooltipV2Expandable.updateSizeAsUIElement(tp);
    tp.setForceProcessInput(true);
    if (mTooltipToHasScrollbar.get(tp) != null && mTooltipToHasScrollbar.get(tp).booleanValue()) {
        ScrollPanel scroll = new ScrollPanel();
        scroll.setContentSize(tp.getWidth(), tp.getHeight());
        scroll.setSize(tp.getWidth() + 5f, mTooltipToHeight.get(tp).floatValue());
        scroll.setMaxShadowHeight(15f);
        scroll.add(tp).inTL(0f, 0f);
        scroll.setUseSimpleShadows(true);
        tooltip.setExternalScroller(scroll);
        return add(scroll);
    }
    tp.setSize(tp.getWidth(), Math.max(tp.getHeight(), mTooltipToHeight.get(tp).floatValue()));
    return add(tp);
}
```
</details>

Lastly there is a method to add a background panel to a tooltip called `wrapTooltipWithBox` for some reason. It creates a `UIPanel` instance with a modified `renderImpl`, where a quad draw call is made. I see this as a pointless addition, as the tooltip itself already has a toggleable background. Its alpha can be modified using the `setBgAlpha` method, and the `TooltipMakerAPI` instance can be cast to `StandardTooltipV2Expandable` and the method `setShowBackground` can be called. Perhaps it is also a good time to mention that vanilla tooltips are actually 10 pixels thinner than the width parameter provided, as the logical gap is filled visually by the borders.

<details>
<summary>Code snippet</summary>

```java
public UIPanelAPI wrapTooltipWithBox(TooltipMakerAPI tooltipMakerAPI, float padLeft, float padRight, float padBelow, float padAbove, final Color color) {
    UIPanel wrapper = new UIPanel(){

        @Override
        protected void renderImpl(float alpha) {
            Position pos = this.getPosition();
            float x = pos.getX();
            float y = pos.getY();
            float w = pos.getWidth();
            float h = pos.getHeight();
            /** quad draw call */
            public.o00000(x, y, w, h, 1f, color, alpha);
            super.renderImpl(alpha);
        }
    };
    wrapper.add(tooltipMakerAPI).inTL(padLeft - 5f, padAbove);
    wrapper.setSize(tooltipMakerAPI.getWidthSoFar() + padLeft + padRight, (tooltipMakerAPI).getHeight() + padBelow + padAbove);
    return wrapper;
}
```
</details>

To get a `CustomPanelAPI` instance through the API, the `SettingsAPI` method `createCustom` can be used.

<br><br>

## **PositionAPI**

Complex layouts are made simpler by giving each UI component a position relative to a parent. The component would then base their positioning on their parent's position. The **Transform Hierarchy** is the thing that lets us create nested ui components to begin with. Could designing a scrollable panel be imaginable without relative positioning? Certainly not. In game UI, each element has a **Transform**, which dictates the position, the scale and the rotation of a ui element. Further, each Transform can have a parent transform for relative transformation or a child transforms to be a parent itself. The **GUI** as implemented by Starsector does not have native support for scale and rotation in its **Transform**, and therefore it is called a `Position`. To scale or rotate children, the **openGL** calls `GL11.glRotated` and `GL11.glScaled` can be called before rendering children. Don't forget to `GL11.glPushMatrix()` before rendering and `GL11.glPopMatrix()` after rendering to keep the transformations local to the component and its children. I digress, **Position Hierarchy** is implemented separately from the **UIPanel Hierarchy** in Starsector. One UI Tree for the panels, and one for their `Position`s. This allows adding only the position of a component to a parent so that it is positioned relative to the parent but manages its traverse methods independently.

`Position` uses relative positioning. As such, it has no `setX` or `setY` methods. Instead, it has relative alignment parameters that describe its transformation relative to its parent. These alignment parameters, together with the calculated coordinates of its parent, allows the `Position` to calculate its own coordinates, which can be retrieved using `getX` and `getY`. Starsector uses the **openGL** convention when it comes to screen space, where the bottom left corner of the screen is <0, 0> and the top right is <1, 1>. The `getX` and `getY` methods return coordinates according to this convention. Further, the origin of the `Position` is its bottom-left corner as well. Therefore when drawing a quad, the bottom left corner would be `(x, y)`, bottom right would be `(x + w, y)`, top right would be `(x + w, y + h)` and top left would be `(x, y + h)`.

The `recompute` method is used to lazily update the coordinates of a `Position`. The update propogates to children, meaning when `recompute` is called on a `Position`, its children also call `recompute`. Further, positions can anchor themselves to other positions. And when the anchor `Position` recomputes its position, this also is propogated to the anchored `Position`s. It should be kept in mind that the recompute function can be quite expensive, so triggering it each frame is a bad idea.

<details>
<summary>Code snippet</summary>

```java
public class Position implements PositionAPI, Cloneable {
/** Cached x coordinate. */
private float mPosX;
/** Cached y coordinate. */
private float mPosY;
/** Logical width of the position used for alignment and rendering. */
private float mWidth;
/** Logical height of the position used for alignment and rendering. */
private float mHeight;
/** Parent of the position used for updating siblings and position. */
private Position mParent;
/** The anchor is used if present instead of the parent for positioning. */
private Position mAnchor;
/** Scalar for the origin's x coordinate by the width of the base. For the value 1, the right side of the base is considered the origin. */
private float mBaseAnchorX;
/** Scalar for the origin's y coordinate by the height of the base. For the value 1, the top side of the base is considered the origin. */
private float mBaseAnchorY;
/** Scalar for the x coordinate by mWidth. For the value -1, the right side of this position is considered the origin. */
private float mSelfAlignX;
/** Scalar for the y coordinate by mHeight. For the value -1, the top side of this position is considered the origin. */
private float mSelfAlignY;
/** Shifts the final x coordinate by the specified amount. Used by inTL or similar and setXAlignOffset. */
private float mAlignOffsetX;
/** Shifts the final y coordinate by the specified amount. Used by inTL or similar and setXAlignOffset. */
private float mAlignOffsetY;
/** An additional and persistent offset set only by setOffset */
private float mOffsetX;
/** An additional and persistent offset set only by setOffset */
private float mOffsetY;
private List<Position> mChildren = new ArrayList<>();
/** When the coordinates or the size of the position is updated, notifies this listener. */
private PositionUpdateListener mUpdateListener;
/** Rounds coordinates after recomputing them. */
private boolean mRoundCoord = true;
/** Prevents the positions from being updated when recompute is triggered. */
private boolean mSuspendRecompute = false;
/** Prevents sibling updates when true. */
public static boolean suspendSort = false;
/** Literally the same thing as suspendSort, but with a different name. */
private boolean mDoSort = true;
....
}
```
</details>

### **The recompute formula**
For each element, `posX` and `posY` (the bottom-left corner) are calculated as:

```java
mPosX = base.mPosX + mBaseAnchorX * base.mWidth + mWidth * mSelfAlignX + mAlignOffsetX + mOffsetX;
mPosY = base.mPosY + mBaseAnchorY * base.mHeight + mHeight * mSelfAlignY + mAlignOffsetY + mOffsetY;
```
- **base**: The reference `PositionAPI` instance. All positioning is relative to the base's bottom-left corner (base.mPosX, base.mPosY).
- **mBaseAnchorX**, `mBaseAnchorY`: Fractions (usually 0 to 1) that define an anchor point on the base's bounding box. Multiplying by the base's width/height converts these fractions into pixel coordinates relative to the base's origin.
    - **mBaseAnchorX** 0 = left edge, 0.5 = horizontal center, 1 = right edge.
    - **mBaseAnchorY**: 0 = bottom edge, 0.5 = vertical center, 1 = top edge.
- **mAlignOffsetX**, **mAlignOffsetY**: Fractions that specify how the element's own bounding box is aligned relative to the anchor point. Multiplying by the element's own width/height gives the necessary shift.
    - **mAlignOffsetX**: 0 = element's left edge at the anchor, -0.5 = element's horizontal center at the anchor, -1 = element's right edge at the anchor.
    - **mAlignOffsetY**: 0 = element's bottom edge at the anchor, -0.5 = element's vertical center at the anchor, -1 = element's top edge at the anchor.
- **mAlignOffsetX**, **mAlignOffsetY**: Pixel offsets added after the alignment calculation. These are the gap/spacing values passes to methods like **inTL(gapX, gapY)**.
- **mOffsetX**, **mOffsetY**: Additional manual offsets, separate from alignment margins. They default to 0 and can be adjusted via **setOffset()**.

Every positioning shortcut like `inTL(x, y)`, `inMid()`, `aboveLeft(anchor, gap)`, etc., simply calls the method:
```java
relativeTo(anchor, baseAnchorX, baseAnchorY, selfAlignX, selfAlignY, AlignOffsetX, AlignOffsetY) /** setting target as null uses parent */
```

Examples
```java
public Position inBR(float offsetX, float offsetY) {
    return relativeTo(null, 1.0f, 0.0f, -1.0f, 0.0f, -offsetX, offsetY);
}

public Position inMid() {
    return relativeTo(null, 0.5f, 0.5f, -0.5f, -0.5f, 0.0f, 0.0f);
}

public Position leftOfTop(UIComponentAPI anchor, float gap) {
    return relativeTo(anchor, 0.0f, 1.0f, -1.0f, -1.0f, -gap, 0.0f);
}
```

Then the `relativeTo` method simply updates the fields and calls `recompute`. Here is the entire recompute method for those interested:

<details>
<summary>Code snippet</summary>

```java
public void recompute() {
    if (mSuspendRecompute) return;
    
    final Position base = mAnchor == null ? mParent : mAnchor;
    if (base != null) {
        final float oldPosX = mPosX;
        final float oldPosY = mPosY;
        mPosX = base.mPosX + mBaseAnchorX * base.mWidth + mWidth * mSelfAlignX + mAlignOffsetX + mOffsetX;
        mPosY = base.mPosY + mBaseAnchorY * base.mHeight + mHeight * mSelfAlignY + mAlignOffsetY + mOffsetY;
        mPosX = mRoundCoord ? Math.round(mPosX) : mPosX;
        mPosY = mRoundCoord ? Math.round(mPosY) : mPosY;

        if (mUpdateListener != null && (oldPosX != mPosX || oldPosY != mPosY)) {
            mUpdateListener.locationChanged();
        }
    }
    if (mChildren.contains(this)) {
        throw new RuntimeException("Children contains this");
    }
    updateSiblings();
    for (Position child : mChildren) {
        child.recompute();
    }
}

private void updateSiblings() {
    if (mSuspendRecompute || !mDoSort || suspendSort) return;
    
    LinkedList<Position> linkedList = new LinkedList<>();
    int counted = 0;
    while (linkedList.size() < mChildren.size()) {
        for (Position child : mChildren) {
            if (linkedList.contains(child)) continue;
            final Position anchor = child.mAnchor;
            if (anchor != null && !mChildren.contains(anchor)) {
                throw new RuntimeException("May only anchor on siblings");
            }
            if (anchor != null && !linkedList.contains(anchor)) continue;
            linkedList.add(child);
        }
        if (++counted <= mChildren.size()) continue;
        throw new RuntimeException("Circular dependency of sibling positions detected");
    }
    mChildren.clear();
    mChildren.addAll(linkedList);
}
```
</details>

<br><br>

## **LabelAPI**

The best way to, and if you are not crazy like LazyWizard, the only way to display text in Starsector is using `LabelAPI`. Used by the game everywhere, even the tooltip uses it to render `String`s. LabelAPI is technically a core component, as it implements `InternalUIComponentAPI` from scratch. Therefore, it cannot have children. I will not pretend to understand how `LabelAPI` renders text, and as such, only its usage will be covered.

The most significant aspect of `LabelAPI` is how it handles fonts. Once the font is specified inside the constructor, it is final and cannot be changed. A new `Label` must be created instead. Secondly, while `Label` does not support direct rendering with specified coordinates, its PositionAPI can nevertheless be used to render it without a parent. To do this, the `PositionAPI` of the label is retrieved, added to the screen panel as a child (only the position) and its xAlignOffset and yAlignOffset are set to the desired coordinates using `setXAlignOffset` and `setYAlignOffset`. The order is significant, and the `PositionAPI` of the label must have a parent or an anchor, otherwise its position won't be recomputed (see section [`PositionAPI`](#positionapi)). After this setup, the `render` method of `LabelAPI` can simply be called.

When first creating a `LabelAPI` instance through the `SettingsAPI` method `createLabel`, the initial width of the label is determined by the first parameter, which constitutes the initial text, and the second parameter, which is the font. The width and height of the `Label` can be adjusted directly through its `PositionAPI`. In general, the render method divides the String characters and renders them in separate lines to ensure their total width do not exceed that of its `PositionAPI`. This system breaks down for extremely thin labels, where the width of the label is less than a character.

The logical positioning of the letters inside the `LabelAPI` is determined by the Alignment parameter, which can be set using the `setAlignment` method. `BL` places the text block to the bottom left side of the `PositionAPI` area, `LMID` puts the text to the left border of the position area, but centers it vertically, and so on. The horizontal positioning only matters if the width of the `LabelAPI` differs from the width of its content, which is usually not the case, as the width of the `LabelAPI` is not usually adjusted after creating it. The same thing applies to vertical positioning and height. For this reason, when there is a need to display a text in the middle of a panel, either the `LabelAPI` can be created and positioned `inMid` of the parent panel, or the `LabelAPI` is resized to cover the entire parent panel and its alignment mode set to `MID`.

The color used to render the text is by default `Global.getSettings().getColor("standardTextColor")`, but it can be modified using `setColor`. Further, certain letters can be highlighted, which basically acts as a secondary color. Technically more than one highlight color can be present. The methods `setHighlight`, `highlightFirst`, `highlightLast` and `unhighlightIndex` can be used to determine what subset of the String is highlighted. `setHighlightColors` can be used alongside `setHighlight(String ... substrings)` to have multiple highlight colors inside the same `LabelAPI`.

The convenience method `autoSizeToWidth` uses the provided width and the cumulative height of the lines resulting from it to calculate the minimum height of the `LabelAPI` and apply it. Useful when stacking lots of `LabelAPI`'s so that they look like a continuous text of paragraphs. Which is exactly what `TooltipMakerAPI` does. The `computeTextWidth` and `computeTextHeight` methods can be treated as static methods, except they utilize the font of the `LabelAPI` the method is called from.

Should `highlightOnMouseover` be true when a `LabelAPI` is hovered, which can be toggled using `setHighlightOnMouseover`, the glow fader's state will be set to IN, which is separate from the **flash** fader, which can be used to make a `LabelAPI` glow using code.

A list of fonts `LabelAPI` accepts can be found under `com.fs.starfarer.api.ui.Fonts`, but there are other fonts not present here too. Should a custom font be used, it must first be loaded using the `SettingsAPI` method `loadFont`.

<br><br>

## **ButtonAPI**

Containing four faders for visual effects, a text label, audio feedback, and a renderer for custom corners, `Button`, which extends `UIComponent` and implements `ButtonAPI`, is one of the most commonly used UI widgets after `LabelAPI` and `TooltipMakerAPI`. 
<details>
<summary>Code snippet</summary>

```java
/** The renderer that dictates the visual appearance. Swapping this changes the button type (text, checkbox, area, image, etc.). */
private BaseButtonRenderer mRenderer;
/** The toggle state for checkboxes or toggle buttons. Inverted on every successful press. */
private boolean mChecked = false;
/** Whether the button accepts input. When false, the button dims and ignores clicks (unless overridden). */
private boolean mEnabled = true;
/** One-shot fader triggered by flash(). Provides a brief burst of brightness, used by keyboard shortcut feedback. */
private Fader mFlashFader;
/** Fades in on mouse hover, out on mouse leave. Creates the standard hover glow. */
private Fader mGlowFader = new Fader(0.0f, 0.05f, 0.25f);
/** Triggers a brief pulse on button down, fading out on release. Provides tactile visual feedback for a click. */
private Fader mButtonPressFader = new Fader(0.0f, 0.05f, 0.25f);
/** The internal state machine and input processor. Handles click detection, shortcut matching, and state transitions. */
private BaseButtonLogic mButtonLogic;
/** Programmatic highlight fader controlled by highlight() and unhighlight(). Used when the button is selected etc. */
private Fader mHighlightFader = new Fader(0.0f, 0.05f, 0.25f);
private float mHighlightBrightness = 0.85f;
/** The ActionListenerDelegate callback. Invoked on a successful button press. */
private InternalActionListenerDelegate mListener;
private String mHoverSound;
private String mPressSound;
private String mPressDisabledSound;
/** Arbitrary data associated with the button. Used as the first argument of the listener method. */
private Object mCustomData = null;
/** If true, suppresses the press sound for the next triggered press. Useful when a parent UI already plays a sound. */
private boolean mSkipPressedSoundOnce = false;
private float mHoverSoundVolume = 1.0f;
private float mPressSoundVolume = 1.0f;
private float mDisabledPressSoundVolume = 1.0f;
/** Independent of mEnabled, input processing is entirely skipped when false. Used to disable interactions without altering visual state. */
private boolean mActive = true;
/** When true, the tooltip attached to this button remains visible even if mActive is false. */
private boolean mShowTooltipWhileInactive = false;
/** When true, right-clicking the button while disabled fires the disabled press logic (sound and optional action). */
private boolean mRightClickWhenDisabled = false;
private float mFlashBrightness = 1.0f;
private float mGlowBrightness = 1.0f;
/** Additive contribution of the flash fader to the final glow value (0.0 = flash replaces glow, 0.2 = flash stacks additively on top). */
private float mFlashToGlowAdditiveRatio = 0.0f;
private boolean mPerformActionWhenDisabled = false;
```
</details>

The `Button` is composed of three parts: the base, the logic, and the renderer. This composite design makes `Button` highly flexible.

In order for the button to be clickable using a shortcut, a shortcut can be assigned using `setShortcut`, where the `key` parameter is an `org.lwjgl.input.Keyboard` constant and the `putLast` parameter appends the shortcut key either to the end of the button text (if true) or highlights the specific key if it appears on the text of the button (if false). This behavior works only when `mRenderer` is a text renderer.

The "quick" mode, which can be toggled using `setQuickMode` modifies the related field inside `mButtonLogic`, which stops the checked state of the button from being modified, i.e. the button cannot be selected, useful for buttons that should not toggle state when pressed. When the clickable state of the button is false, toggled using `setClickable`, the button does not register mouse clicks, but keyboard shortcuts and programmatic clicks work. When `isEnabled` is false, the button skips `processInputImpl` and also doesn't call the listener if additionally it isn't a right click and `mRightClickWhenDisabled` is true. The renderer also receives the enabled state of the button as a parameter. The checked state of the button, used by the renderer and `buttonPressed` button, can be retrieved using `isChecked` and toggled using `setChecked`. The state also gets toggled inside `buttonPressed`.
<details>
<summary>Code snippet</summary>

```java
@Override
public void buttonPressed(InputEventAPI event, Object source) {
    if (!(mEnabled || mRightClickWhenDisabled && event.isRMBEvent())) {
        if (mPressDisabledSound != null) {
            Global.getSoundPlayer().playSound(mPressDisabledSound, 1.0f, 1.0f * mDisabledPressSoundVolume);
        }
        if (event != null) {
            event.isKeyboardEvent();
        }
        flash(false);
        mButtonPressFader.fadeOut();
        if (mPerformActionWhenDisabled && mListener != null) {
            mListener.actionPerformed(event, source);
        }
        return;
    }
    if (mPressSound != null) {
        if (mSkipPressedSoundOnce) {
            mSkipPressedSoundOnce = false;
        } else {
            Global.getSoundPlayer().playSound(mPressSound, 1.0f, 1.0f * mPressSoundVolume);
        }
    }
    mChecked = !mChecked;
    mButtonPressFader.fadeOut();
    if (mListener != null) {
        mListener.actionPerformed(event, source);
    }
    if (event.isKeyboardEvent() && (mFlashFader == null || mFlashFader.getBrightness() < 0.7f)) {
        flash(false);
    }
}
```
</details>

The `getGlowAmount` method serves as the central point for all of the button's visual feedback, combining the states of the four faders into a single brightness value passed to the renderer. By default, `mGlowFader` and `mFlashFader` operate in a mutually exclusive manner, where the **base** intensity is determined by the brightest one. `mHighlightFader` acts as an override, modified with `highlight` and `unhighlight`, overriding the **base** if its brightness exceeds it, making external programmatic brightness always dominant. This default behavior changes when `mFlashToGlowAdditiveRatio` is non-zero value, which can be set using the `setAddFlashToGlow` method, missing from `ButtonAPI`. In this additive mode, the flash is an overlay; where the **base** intensity is taken exclusively from `mGlowFader`, and a portion of the flash's brightness (scaled by the ratio) is added on top, capped at `1f` to prevent oversaturation. Lastly, the method applies a subtle blend with `mButtonPressFader` when the button is not actively being held down, contributing 33% of `mButtonPressFader`'s brightness to the final value to produce a pulse after a click.
<details>
<summary>Code snippet</summary>

```java
/** The method is modified for readability, but the logic is unaltered. */
public float getGlowAmount() {
    final float glow = mGlowFader.getBrightness() * mGlowBrightness;
    final float highlight = mHighlightFader.getBrightness() * mHighlightBrightness;
    float flash = mFlashFader != null ? mFlashFader.getBrightness() * mFlashBrightness : 0f;
    if (mFlashToGlowAdditiveRatio > 0f) flash *= mFlashToGlowAdditiveRatio;
    
    float base = mFlashToGlowAdditiveRatio > 0f ? glow : Math.max(glow, flash);
    base = Math.max(base, highlight);

    if (mFlashToGlowAdditiveRatio > 0f && flash > 0f) {
        base = Math.min(1f, base + flash);
    }
    if (!mButtonLogic.isPressed()) {
        base = 0.67f * base + mButtonPressFader.getBrightness() * 0.33f;
    }
    return base;
}
```
</details>

When managing the tooltip, `mShowTooltipWhileInactive` is not used by `ButtonAPI` anywhere, as it already extends `UIComponent`, which handles the tooltip lifecycle. The button can programmatically be brightened using the `flash` methods. When the listener gets called can also be controlled using `setPerformActionWhenDisabled`. And the click sound can be skipped for once using `setSkipPlayingPressedSoundOnce`. Custom data about the button can be set using `setCustomData`. Should `setHighlightBounceDown` be true, then the highlight fader will bounce down once it reaches max brightness.

<br><br>

## **Fader**

A `Fader` is a utility class that interpolates a brightness value between `0.0f` (fully hidden) and `1.0f` (fully visible) over time. It is used throughout the UI to drive visual transitions such as hover glows, button presses, flashes, and fade-in/fade-out animations. `FaderUtil` is functionally identical.

The fader is a state machine with three states defined by the `State` enum:
- **IN** – Brightness advances toward `1.0f` over the duration specified by `durationIn`.
- **OUT** – Brightness declines toward `0.0f` over the duration specified by `durationOut`.
- **IDLE** – Brightness remains frozen at its current value.

The lifecycle is driven by repeated calls to `advance(float delta)`, which updates the brightness based on the current state and elapsed time. When the brightness reaches an extreme (`0.0f` for `OUT`, `1.0f` for `IN`), the fader either transitions to `IDLE` or, if bounce mode is enabled, automatically reverses direction. Bounce behavior is controlled by two flags: `bounceUp` (switches from `OUT` to `IN` when brightness reaches `0`) and `bounceDown` (switches from `IN` to `OUT` when brightness reaches `1.0f`).

The fader provides convenience methods to control its state. `fadeIn` and `fadeOut` initiate transitions toward full visibility or full concealment, respectively. If the duration is `0.0f`, these methods immediately force the brightness to the target value via `forceIn` or `forceOut`, bypassing the animation. The current brightness can be queried with `getBrightness`, and the fader's state can be inspected using `isFadingIn`, `isFadingOut`, `isIdle`, `isFadedIn`, and `isFadedOut`.

<br><br>

## **TooltipMakerAPI**

The most extensive UI element of them all, `TooltipMakerAPI` is really 5 different UI elements inside a trench coat. Not only is it used for creating tooltips, but it is also used for panel creation. Buttons, grids, images, icons, tables, cargo, ship list, checkboxes, text fields, maps, labels, planet info, and the skill panel, `TooltipMakerAPI` has it all. One other advantage is the automatic UI layouting, where each added element is placed below the previously added element with relative positioning. Further, the implementation class of `TooltipMakerAPI` is not obfuscated in name, thus it can be addressed without reflection. `StandardTooltipV2Expandable` implements `TooltipMakerAPI` and extends `StandardTooltipV2`, which itself extends `UIPanel`.

It is important to note that `StandardTooltipV2Expandable` is a very memory-expensive UI element. It has many fields, and even fields that point to other `StandardTooltipV2Expandable`s. Therefore, I heavily advise you against spamming `TooltipMakerAPI`s, as that can very quickly tank performance. One common usage of the tooltip is creating buttons. But each button doesn't need its own tooltip. Instead, since the buttons are their own `UIPanel`, one `TooltipMakerAPI` can be used to create many `ButtonAPI`s, and they can be attached to a different `UIPanel` or positioned later. As such, `StandardTooltipV2Expandable` can sometimes act as a glorified factory class with an instance.

This 5300 line monolith will take some time to cover, and cannot be complete, as I don't have the time for that. There are two ways to use `TooltipMakerAPI`: as a tooltip or as a UI builder. UI building will be covered first. But more importantly, the fields list. Types are de-obfuscated, some names were not obfuscated to begin with. Added comments wherever necessary.

<details>
<summary>Code snippet</summary>

```java
public abstract class StandardTooltipV2Expandable extends StandardTooltipV2 implements TooltipMakerAPI {
    /** Content panel of the tooltip. */
    protected UIPanel panel = null;
    /** The last element inserted to the content panel. Not every element is assigned, such as currTable or currGrid. */
    protected InternalUIComponentAPI prev = null;
    /** The tracked height of the tooltip, which can be modified using setHeightSoFar. */
    protected float height = 0f;
    /** The with of the tooltip itself, which almost all elements use as their width, such as addPara or addGrid.  */
    protected float width = 350f;
    /** The font used by the addTitle method. */
    protected String titleFont = Fonts.ORBITRON_12;
    /** Used by addPara and createLabel. */
    protected String paraFont = Fonts.DEFAULT_SMALL;
    /** Used by addAreaCheckbox. */
    protected String areaCheckboxFont = Fonts.DEFAULT_SMALL;
    protected String gridFont = Fonts.DEFAULT_SMALL;
    protected Color titleFontColor = Global.getSettings().getColor("tooltipTitleAndLightHighlightColor");
    protected Color paraFontColor = Global.getSettings().getColor("standardTextColor");
    /** Used internally as a state tracker and as a flag to use a preset font. Honestly just modify titleFont. */
    protected boolean titleSmallOrbitron = false;
    /** Same as titleSmallOrbitron. Prefer using paraFont. */
    protected boolean paraSmallInsignia = false;
    /** Same as above. */
    protected boolean paraSmallOrbitron = false;
    /** Straight up not used by anything. Why is this here? */
    protected boolean gridSmallInsignia = false;
    private boolean recreateEveryFrame = false;
    protected ModGrid currGrid = null;
    /** Think of this as the row count instead of the height in pixels. So multiply this by gridRowHeight for the actual height. */
    private int currGridHeight = 0;
    private float gridRowHeight = 15f;
    /** Used to create stacked commodity icons, used by the industry tooltip for example. */
    private IconGroup currIconGroup;
    /** Used by beginImageWithText to hold the image and the text. */
    private UIPanel imageContainer = null;
    /** Used by beginImageWithText. I can guarantee that it does something. */
    private StandardTooltipV2Expandable imageWithTextTp = null;
    /** Same as above. */
    private InternalUIComponentAPI imageContainerTp = null;
    /** beginImageWithText uses this as the imageContainer width. */
    private float beginImageWithTextWidth = 0f;
    private boolean beginImageWithTextMidAlignImage = true;
    /** Used internally and I don't care how it works */
    private UIPanel beginCustomWithCustom_panel1 = null;
    /** Used internally and I don't care how it works */
    private StandardTooltipV2Expandable beginCustomWithCustom_tooltip1 = null;
    /** Used internally and I don't care how it works */
    private InternalUIComponentAPI beginCustomWithCustom_inputPanel = null;
    /** Used by beginSubTooltip and endSubTooltip. Does not get reset after endSubTooltip is called. */
    private StandardTooltipV2Expandable subTooltip = null;
    /** Determines text label prefix text color if bulletListMode is not null. */
    private Color bulletColor = null;
    /** Determines text label prefix text width if bulletListMode is not null. */
    private Float bulletWidth = null;
    /** Prefix string for paragraphs. */
    private String bulletListMode = null;
    /** Used by addPara and createLabel. */
    private float textWidthOverride = 0f;
    /** Provided to buttons created by this tooltip as the action listener. The SAM is actionPerformed. */
    private ActionListenerDelegate buttonListener;
    private String buttonFont = Fonts.ORBITRON_12;
    /** You will never use this. Irrelevant */
    private UIPanel parentWidget = null;
    private UITable currTable = null;
    public static float DEFAULT_REL_BAR_WIDTH = 128f;
    /** The intel panel is passed from the CustomPanelAPI that created this tooltip if it had any to begin with. */
    private IntelUIAPI intelPanel;
    /** The tooltip is placed inside the content panel of the scroll panel when adding it using addUIElement. */
    protected ScrollPanelAPI externalScroller = null;
}

public class StandardTooltipV2 extends UIPanel implements ..., Cloneable {
    /** Literally the same content panel as StandardTooltipV2Expandable panel. */
    private final InternalUIComponentAPI contentPanel;
    /** Used internally by createWeaponTooltip and nothing else. */
    private UIPanel customPanel;
    /** Left visual border panel. */
    protected BorderRenderer left;
    /** Right visual border panel. */
    protected BorderRenderer right;
    protected float bgAlpha = 0.85f;
    protected Runnable beforeShowing = null;
    private Runnable beforeExpanding = null;
    private boolean isExpandable = false;
    protected boolean expanded = false;
    /** Opens a dialog window when expanded as far as I can tell. */
    private boolean isExpandAsDialog = true;
    /** Default value is F1. */
    protected String expandHotkeyStr = null;
    /** When the codex is opened, this id is used to display content. */
    protected String codexEntryId = null;
    /** Not used by anything anywhere. */
    protected String[] expandHotkeyArray = null;
    /** Prevents the tooltip from expanding. */
    protected boolean expandableCodexOnly = false;
    /** Used when isExpandable is true. */
    protected String expandHotkeyStr2 = null;
    /** Used by createCostLabel method. */
    protected FleetMemberAPI codexFleetMember = null;
    /** I don't know why it's called temporary. It is not reset after getting displayed. */
    protected CodexEntryPlugin tempCodexEntry = null;
    /** The logical UIComponent that manages the lifecycle of the tooltip. */
    private InternalUIComponentAPI ownerWidget = null;
    protected String expandString = "Press %s for more info";
    protected String unexpandString = "Press %s to hide";
    /** The gap between the tooltip horizontal sides and the content panel. Default is 5f. */
    private float outerPadX = 0f;
    /** The gap between the tooltip vertical sides and the content panel. Default is 0f. */
    private float outerPadY = 0f;
    /** Processes input events even when invisible. */
    private boolean forceProcessInput = false;
    /** Does not detach the tooltip when the mouse is over it. */
    private boolean forceShowWhileMouseInside = false;
    /** Custom logic when the tooltip is expanded. */
    protected Runnable onExpandInsteadOfExpand = null;
    /** Removes itself when the fader brightness is 0. */
    protected boolean selfRemove = true;
    /** The borders on the horizontal sides. */
    private boolean showBorder = true;
    /** The label at the bottom of the tooltip that holds the expandString and the unexpandString. */
    protected Label expandLabel;
    /** Simple black quad. Alpha multiplied with bgAlpha. */
    private boolean showBackground = true;
    /** Used by expandInPlace. */
    protected float savedX;
    /** Used by expandInPlace. */
    protected float savedY;
    /** Internal state tracker. */
    private boolean isBeingShown = false;
    /** Used by processInputImpl when isExpandAsDialog is true. Serves as the input event detector panel.  */
    private InputInterceptor someInternalInterceptionPanel;
    /** Tooltip logic delegate. */
    private TooltipLogic logicDelegate = null;
    /** If false, places the tooltip in the middle of the screen. Always true for StandardTooltipV2Expandable. */
    private boolean expandInPlace;
}
```
</details>

### **Managing the UI builder**
The best way, and as far as I am aware, the only way, to use `TooltipMakerAPI` as a UI builder is through the `CustomPanel`, which has methods for creating a tooltip (`createUIElement`) and adding that tooltip properly to the panel itself (`addUIElement`). Please keep in mind that the tooltip added to a `CustomPanel` must also be created by the same custom panel. Otherwise it won't know the height of the tooltip or whether if it should have a scrollbar. Basically it will break. For more details see section `CustomPanelAPI`.

The tooltip does not place its content inside itself, which it could, because it extends `UIPanel`, but has a separate content panel. Therefore, the children interaction methods from `UIPanelAPI` such as `addComponent` or `removeComponent` should not be used when working with `TooltipMakerAPI`. There is no API side access to the content panel, however, `StandardTooltipV2Expandable` has a getter for the content panel not present on `TooltipMakerAPI`, which can be called using reflection. The method signature is `UIPanel getPanel()`. Note the replacement of the obfuscated type with `UIPanel`, hence the need for reflection.

Should the tooltip have a scrollbar, it will be placed inside a `ScrollPanel` (see section [`CustomPanelAPI`](#custompanelapi)), which can be retrieved through `TooltipMakerAPI` using `getExternalScroller`. The stored external scroller instance can also be changed using `setExternalScroller`, but it doesn't actually change the scroll panel the tooltip is attached to.

### **Managing the tooltip**
The traditional tooltip, which gets managed by the `UIComponent` it gets attached to (see section [`UIComponentAPI`](#uicomponentapi)), cannot be directly instanced through `TooltipMakerAPI`. And creating one by subclassing `StandardTooltipV2Expandable` is not worth the extra hassle, unless some extra setup methods only available through `StandardTooltipV2Expandable` are called. Instead, the better way to define a tooltip is done using the `TooltipCreator` interface, and can be positioned using the TooltipLocation enum. It's background color can also be changed using `setBgAlpha`.
```java
public interface TooltipCreator {
    boolean isTooltipExpandable(Object tooltipParam);
    float getTooltipWidth(Object tooltipParam);
    void createTooltip(TooltipMakerAPI tooltip, boolean expanded, Object tooltipParam);
}

public static enum TooltipLocation {
    LEFT,
    RIGHT,
    ABOVE,
    BELOW;
}
```

The implementation of this tooltip can then be added either to the last added `UIComponent` or a specified component. When specifying a `UIComponentAPI` that will manage the tooltip, the method can be treated as static, as it uses no members.
```java
/** recreateEveryFrame is true by default */
void addTooltipToPrevious(TooltipCreator tc, TooltipLocation loc);
void addTooltipToPrevious(TooltipCreator tc, TooltipLocation loc, boolean recreateEveryFrame);
/** recreateEveryFrame is true by default */
void addTooltipTo(TooltipCreator tc, UIComponentAPI to, TooltipLocation loc);
void addTooltipTo(TooltipCreator tc, UIComponentAPI to, TooltipLocation loc, boolean recreateEveryFrame);
```

There are additional methods for adding tooltips to table headers and rows.
```java
void addTableHeaderTooltip(int colIndex, TooltipCreator tc);
void addTableHeaderTooltip(int colIndex, String text);
void addTooltipToAddedRow(TooltipCreator tc, TooltipLocation loc);
/** recreateEveryFrame is true by default */
void addTooltipToAddedRow(TooltipCreator tc, TooltipLocation loc, boolean recreateEveryFrame);
```

### **Using the tooltip - general**
All elements that are added to the tooltip, without explicitly using `addCustomDoNotSetPosition`, ultimately get positioned using the `addCustom` method, or in a way that is similar to `addCustom`, which positions all elements relative to the last added element, which can be acquired using the `getPrev` method. The tracked height of the tooltip is also updated alongside the positioning, which removes the layout burden from the modder when working with a `TooltipMakerAPI`. It should be noted that the horizontal alignment (x-axis) also follows the last added element.

<details>
<summary>Code snippet</summary>

```java
public UIComponentAPI addCustomDoNotSetPosition(UIComponentAPI comp) {
    /** Just adds the component to the content panel. */
    this.panel.add(comp);
    return comp;
}

public UIComponentAPI addCustom(UIComponentAPI comp, float vGap) {
    if (prev != null) {
        panel.add(comp).belowLeft(prev, vGap);
    } else {
        panel.add(comp).inTL(5f, vGap);
    }
    prev = comp;
    height += (comp).getHeight() + vGap;
    return comp;
    }
```
</details>

After creating a tooltip, you might want to add a title to your section. `TooltipMakerAPI` has a method just for that, which creates a `LabelAPI` and places it at the top of left side of the tooltip content panel. For this reason, the title should either be the first thing placed inside the tooltip, or it should not be placed at all, as it does not get positioned relative to the previous element. When adding a title with `addTitle`, a color can be provided instead of the default `titleFontColor`. Note that the width of the `LabelAPI` is equal to the width of the tooltip itself. The default title font can be modified using `setTitleFont` (or `setTitleSmallOrbitron`) and the default title color using `setTitleFontColor`.

<details>
<summary>Code snippet</summary>

```java
public Label addTitle(String string) {
    return addTitle(string, null);
}

public Label addTitle(String string, Color color) {
    Label titleLabel = new Label(string, this.titleFont, this.titleFontColor, true, Alignment.LMID);
    if (this.titleSmallOrbitron) {
        /** Does some extra setup besides the font. */
        titleLabel = Label.createSmallOrbitronLabel(string, Alignment.LMID);
        titleLabel.setColor(this.titleFontColor);
    }
    if (color != null) {
        titleLabel.setColor(color);
    }
    titleLabel.autoSizeToWidth(this.width);
    this.prev = titleLabel;
    /** Always adds it to the top left with a pad of 5, effectively inTL(5f, 0f) */
    this.panel.add(titleLabel).inTL(0.0f, 0.0f).setXAlignOffset(5.0f);
    this.height += titleLabel.getHeight();
    return titleLabel;
}
```
</details>

The first step after adding a title is probably some text. A block of text, if you will. For this, an array of paragraph methods are available. There are a total of five methods for this.

The first method simply adds the text and places it below the previous element with a gap, specified as the `pad` parameter. Indeed, the `pad` parameter serves the same purpose for all `addPara` methods. The default paragraph color is used, which can be modified using `setParaFontColor`.
```java
LabelAPI addPara(String str, float pad);
```

The second `addPara` methods allows a color to be specified, which gets used instead of `paraFontColor`.
```java
LabelAPI addPara(String str, Color color, float pad);
```

The third `addPara` method takes in a formattable string, the vertical gap, the singular highlight color and the array of `String`s to be highlighted. The String formatting is handled by `String.format` and the highlighting is handled by the internal `LabelAPI` renderer.
```java
LabelAPI addPara(String format, float pad, Color hl, String... highlights);
```

The fourth `addPara` method is similar to the third, except it also allows the selection of a custom color for the label.
```java
LabelAPI addPara(String format, float pad, Color color, Color hl, String ... highlights);
```

The last `addPara` method can be used to provide multiple highlight colors for highlighting.
```java
LabelAPI addPara(String format, float pad, Color[] hl, String ... highlights);
```

It should be noted that the highlight colors do not support enumerated formatting, which `String.format` supports. For example, the string "I am going to pay %2$s credits for %1$s." will be formatted as "I am going to pay 40 credits for organics." when the parameters are ["organics", "40"]. But suppose `addPara` is to be used, where the formatting keys are ordered. The provided strings will obey the order, but the provided colors will not. The colors will bind to the formatting keys in the order they appear. And when the formatting keys are enumerated, the colors do not bind to each instance of that formatting key, but only to the first. In the below example, the text is formatted correctly, but the colors are bound only to the first formatting key, independent of its number or how many times it appears.
```java
addPara(
    "%1$s + %3$s: 5  •  %2$s + %3$s: 10  •  %1$s + %2$s + %3$s: 50",
    opad*2,
    Misc.getHighlightColor(),
    "Ctrl", "Shift", "Click"
);
```

The format output is `"Ctrl + Click: 5  •  Shift + Click: 10  •  Ctrl + Shift + Click: 50"`, but the colors are mangled. Only the first appearance of "Ctrl", "Shift" and "Click" are highlighted.

If you are feeling fancy or want to user markup, `addParaWithMarkup` can be used, similar to `addPara`, but with built-in support for inline formatting. It allows the insertion of dynamic values using %s placeholders and color specific substrings directly within the text using the `{{color:colorName|text}}` syntax. Under the hood, it parses this markup and applies the appropriate highlights and Misc.getHighlightColor() to the generated `LabelAPI`. The method uses the default `paraFont`.
```java
LabelAPI addParaWithMarkup(String str, float pad);
LabelAPI addParaWithMarkup(String str, Color color, float pad);
LabelAPI addParaWithMarkup(String str, float pad, String... tokens);
LabelAPI addParaWithMarkup(String str, Color color, float pad, String... tokens);

/** Example usage from game files, where "Ctrl-S" is highlighted. */
addParaWithMarkup("Store the current planet filter settings so they can be loaded later. Can also press {{Ctrl-S}}.", 0f);
```

There are default fonts for paragraphs and titles, which can be toggled using a list of handy methods.
```java
void setTitleOrbitronLarge();
void setTitleOrbitronVeryLarge();

void setParaSmallOrbitron();
void setParaFontOrbitron();
void setParaFontVictor14();
void setParaFontDefault();
void setParaOrbitronLarge();
void setParaOrbitronVeryLarge();
void setParaInsigniaLarge();
void setParaInsigniaVeryLarge();
void setParaSmallInsignia();
```

Also I don't know who needs to hear this, but a `LabelAPI` can be created directly without adding it to the content panel using `createLabel`.

Sometimes a section of the tooltip needs to be visually separated from the rest, or simply a new section must be made obvious. This is where the section heading comes in handy, which is a `LabelAPI` with a colored background. Its `Alignment` can be customized, but usually `MID` is used the most often. The pad parameter, just like most other pad parameters inside the tooltip, specify its vertical gap to the last added element. By default, the width of the `LabeLAPI` similar to how titles and paragraphs work, is equal to the tooltip width, but that can be modified. The usual bgColor is `FactionAPI.getDarkUIColor()` if there is a faction context, `Misc.getDarkPlayerColor()` otherwise.
```java
LabelAPI addSectionHeading(String str, Alignment align, float pad);
LabelAPI addSectionHeading(String str, Color textColor, Color bgColor, Alignment align, float pad);
LabelAPI addSectionHeading(String str, Color textColor, Color bgColor, Alignment align, float width, float pad);
```

One of the most useful UI elements without debate is the button. There is a button factory class in the Starsector API, however, it is obfuscated. Thus, one of the best ways to create a button is through `TooltipMakerAPI`, even though the button has nothing to do with the `TooltipMakerAPI` instance it was created from. For more details on how buttons work, see section [`ButtonAPI`](#buttonapi).
```java
ButtonAPI addButton(String text, Object data, float width, float height, float pad);
ButtonAPI addButton(String text, Object data, Color base, Color bg, float width, float height, float pad);
ButtonAPI addButton(String text, Object data, Color base, Color bg, Alignment align, CutStyle style, float width, float height, float pad);
```

Checkboxes, which use two sprites, one for toggled and one for not toggled, to indicate their state, can also be created using the tooltip. Note that they have no text and are simple sprite renderers with state, fader and audio feedback.
```java
ButtonAPI addCheckbox(float width, float height, String text, Object data, UICheckboxSize size, float pad);
ButtonAPI addCheckbox(float width, float height, String text, Object data, String font, Color textColor, UICheckboxSize size, float pad);
```

An `AreaCheckbox` is a checkbox variant where the entire bounding rectangle (including the text label) serves as the interactive hitbox, meaning clicking anywhere within the designated area toggles the state rather than just the small checkmark square. It offers control over the colors for the unselected state, the hover/selected state, and the text itself.
```java
void setAreaCheckboxFont(String areaCheckboxFont);
ButtonAPI addAreaCheckbox(String text, Object data, Color base, Color bg, Color bright, float width, float height, float pad);
ButtonAPI addAreaCheckbox(String text, Object data, Color base, Color bg, Color bright, float width, float height, float pad, boolean leftAlign);
```

The text font used for the button cannot be changed after creating the `ButtonAPI` instance. Therefore, the button text font can be set using utility methods exposed by `TooltipMakerAPI`.
```java
void setButtonFontDefault();
void setButtonFontVictor10();
void setButtonFontVictor14();
void setButtonFontOrbitron20();
void setButtonFontOrbitron20Bold();
void setButtonFontOrbitron24();
void setButtonFontOrbitron24Bold();

void setAreaCheckboxFontDefault();
```

When the button is clicked, a listener gets called (if provided), so that the logic of the button can be modified without subclassing the button. The source is the `ButtonAPI` itself, and the data is the custom data stored by the button (if any).
```java
public static interface ActionListenerDelegate {
    void actionPerformed(Object data, Object source);
}

/**
 * Needs to be called *before* any methods that create UI elements 
 * that call the action listener (such as addButton) are called.
 * Warning: If the TooltipMakerAPI already has an action listener, it will be overridden.
 */
void setActionListenerDelegate(ActionListenerDelegate delegate);
```

When a set of sets needs to be displayed in a **GUI**, the first option is a table. `TooltipMakerAPI` supports text-only tables, even though the underlying `UITable` class supports any abstract `UIComponentAPI` as a table cell. There are four `beginTable` methods, which return the newly created table, wherein column texts and widths are specified. This is the only place where the internal `currTable` field can be accessed.
```java
/**
 * Columns are pairs of <string name> <Float|Integer width>
 */
UIPanelAPI beginTable(FactionAPI faction, float itemHeight, Object ... columns);
UIPanelAPI beginTable2(FactionAPI faction, float itemHeight, boolean withBorder, boolean withHeader, Object ... columns);
UIPanelAPI beginTable(Color base, Color dark, Color bright, float itemHeight, Object ... columns);
UIPanelAPI beginTable(Color base, Color dark, Color bright, float itemHeight, boolean withBorder, boolean withHeader, Object ... columns);
```

Rows are added to the table using the `addRow` and `addRowWithGlow` methods. The cells only support text data. Tooltips can be added to the last inserted row using `addTooltipToAddedRow`. Similarly, an id can be assigned to the last inserted row using `setIdForAddedRow`, which can be used to identify rows by the listener.
```java
/**
 * Possible sets of data for a column:
 * string |
 * color, string |
 * alignment, color, string |
 * alignment, color, LabelAPI
 */
Object addRow(Object ... data);
Object addRowWithGlow(Object ... data);
```

The table, once all the rows are inserted, can be added to the tooltip using `addTable`, where the `andMore` parameter appends the " ... and {{ andMore }} more" text to the end of the table as a row and the `pad` parameter specifies the gap between the previous element and the table.

The `makeTableItemsClickable` method assigns to each row the `buttonListener`, which can be set using the `setActionListenerDelegate` method. As such, the `buttonListener` must be set before calling this method. The first parameter "`data`" of the `actionPerformed` method is of type `TableRowClickData`, should the event originate from a `UITable` row. `TableRowClickData` contains two public fields: `rowId` and `table`.

Modifiable values, stored using a `MutableStat`, are usually displayed using a `ModGrid`, or simply `Grid` (the class name is obfuscated), within a tooltip. `TooltipMakerAPI` can have at most one grid being worked on, stored by the `currGrid` field.

The default `gridRowHeight` of 15 pixels can be modified using `setGridRowHeight`, `setLowGridRowHeight` (12 px) or `resetGridRowHeight` (15 px). This value is used for layouting. The font used by the grid can be modified with `setGridFont`, `setGridFontDefault` or `setGridFontSmallInsignia`. The color of the value label is set using `setGridValueColor` and the color of the text label can be set with `setGridLabelColor`.

There are several convenience methods to create a grid out of a set of data, be it a `MutableStat`, `StatBonus`, or `CampaignFleetAPI`. The `pad` in the context of these methods is the gap between the last added element and the newly added grid. The `valuePad` is the gap between the value and the text labels. `valueWidth` is a subset of `width`, meaning the width must account for `valueWidth`. The interfaces `StatModValueGetter` and `FleetMemberValueGetter` can be used to define custom formatting and coloring for values.
<details>
<summary>Code snippet</summary>

```java
void addStatModGrid(float width, float valueWidth, float valuePad, float pad, MutableStat stat);
void addStatModGrid(float width, float valueWidth, float valuePad, float pad, StatBonus stat);
void addStatModGrid(float width, float valueWidth, float valuePad, float pad, MutableStat stat, StatModValueGetter getter);
void addStatModGrid(float width, float valueWidth, float valuePad, float pad, StatBonus stat, StatModValueGetter getter);
void addStatModGrid(float width, float valueWidth, float valuePad, float pad, StatBonus stat, boolean showNonMods, StatModValueGetter getter);
void addStatModGrid(float width, float valueWidth, float valuePad, float pad, MutableStat stat, boolean showNonMods, StatModValueGetter getter);
void addStatGridForShips(float width, float valueWidth, float valuePad, float pad, CampaignFleetAPI fleet, int maxNum, boolean ascending, FleetMemberValueGetter getter);
```
</details>

To add a grid with custom values, that is, without providing a data structure, the methods beginGrid and beginGridFlipped can be used. The utility methods `addStatModGrid` and `addStatGridForShips` always flip the grid. The `cols` parameter is used to divide the grid into equally large columns. Flipping a grid places the value label to the left of the text label, whereas the default grid places the value label to the right of the text label. `addToGrid` is used to add cells to the active grid, where the `x` and `y` are integers that determine the cell the value/text pair is to be placed on. The coordinate `<0, 0>` represents the top left cell. The `x` parameter is bounded by `cols - 1`. The parameters can be thought of as `(itemWidth / cols)` and `gridRowHeight` multipliers (excluding padding).
<details>
<summary>Code snippet</summary>

```java
void beginGrid(float itemWidth, int cols);
void beginGrid(float itemWidth, int cols, Color labelColor);
Object addToGrid(int x, int y, String label, String value);
Object addToGrid(int x, int y, String label, String value, Color valueColor);
void beginGridFlipped(float itemWidth, int cols, float valueWidth, float valuePad);
void beginGridFlipped(float itemWidth, int cols, Color labelColor, float valueWidth, float valuePad);
```
</details>

Once all the cells are added to the grid, it can be added to the tooltip using `addGrid`, where the `pad` parameter is the gap between the last added element and the active grid. The currently active grid can be discarded using `cancelGrid`.

If an image needs to be displayed as a panel, the `addImage` methods can be used, where the sprite is wrapped around a simple panel. Multiple images can be displayed with `addImages`. To place a label next to an image, the `beginImageWithText` (2 variants) is called with a sprite id, and the `addPara` method is used. To finish the image + label row, the `addImageWithText` method is called with the gap between the last placed element and the image + label specified.

The most recognizable usage of the icon group is inside the industry production tooltip, where the outputs are displayed as an icon group. The icon grouper supports any sprite, and not just cargo sprites, but the API only accepts `CommoditySpecAPI`s. After calling `beginIconGroup`, each unique commodity is added to the icon group using `addIcons`, where the `num` parameter specifies the amount of sprites to be displayed closely. Each icon section added by `beginIconGroup` also have additional padding between them. Lastly, `addIconGroup` is used to add the icon grouper panel to the tooltip.

### **Using the tooltip - specialized**
Aside from common utilities provided by `TooltipMakerAPI`, there are more situational, but just as useful, methods that the tooltip provides. This list is not complete, and for all the API methods see `com.fs.starfarer.api.ui.TooltipMakerAPI`.

In order that the labels added by `addPara` or similar do not stretch the entire length of the tooltip, the `setTextWidthOverride` can be used.

The player can be provided a field to type using `addTextField`, but the listener used by `TextFieldAPI` is not present in the API, thus the field must be queried each frame for changes in the text.

The `addSpacer` method adds an empty panel with the specified height to create space for the element to be added after it.

`setBulletedListMode` and related methods can be used to include a `String` prefix for labels added by `addPara` and similar.

`String`s can be modified using the convenience methods `shortenString` and `computeStringWidth` (in px).

To add a value + text pair, the `addLabelledValue` method can be used.

The tracked height of the tooltip can be modified using `setHeightSoFar` and retrieved using `getHeightSoFar`. Similar for the width using `getWidthSoFar`, but there is no setter for `width`.

To draw a simple frame with adjustable thickness, the utility method `createRect` can be called and its dimensions and position specified through its `PositionAPI`. The created rect must be added manually to the tooltip.

A new tooltip can be started using `beginSubTooltip`, which simply returns a new tooltip without adding it to the origin tooltip. The `endSubTooltip` simply updates its dimensions and returns it, so it must be added manually to the original tooltip.

In order for the tooltip to open the codex when **2** is pressed, the `setCodexEntryId` method can be used, where the Id is acquired through `CodexDataV2`. Additional codex methods include `setCodexEntryFleetMember`, where a codex entry for a specific variant of a ship is generated, and `setCodexTempEntry`, which is not temporary at all, but instead is displayed before `codexEntryId` if it is not null. It is never reset. A list of codex entry panels can be added to the tooltip using `addCodexEntries`.

A `CargoAPI` can be displayed with `showCargo` methods and a list of `FleetMemberAPI`s can be displayed using `showShips` methods. A list of ships, different from `showShips` in a way I don't know, can be added using `addShipList`.

A list of resources/credits with the available amount (inside the player fleet `CargoAPI`) can be displayed using `showCost` methods. The cost breakdown for surveying is displayed with `showFullSurveyReqs`. Information about a planet, including a planet/station renderer, can be displayed as a panel using `showPlanetInfo` methods, which take in `PlanetInfoParams`.

The relationship bar for a `PersonAPI`, a `FactionAPI`, or simply a value is created, automatically added and positioned using `addRelationshipBar` methods.

Information regarding a story point usage is added with `addStoryPointUseInfo` methods.

The map of the sector can be created using `createSectorMap` and added using `addSectorMap`.

The skills of a `PersonAPI` can be displayed using `addSkillPanel` or `addSkillPanelOneColumn`.



## **Other Interfaces (Roadmap)**

The following sections are planned for future updates to this resource. They are built on the same fundamentals covered here and are not required reading for basic UI work.
- `InputEventAPI` – Coming soon.
- `DialogPanelAPI` – Coming soon.
- `ScrollPanelAPI` – Coming soon.
- `TextFieldAPI` – Coming soon.

<br><br>

# **Working with the GUI**

<br>

## **The Building Block**

The only `UIComponentAPI` instance available through the API is the `CustomPanelAPI`, through `Global.getSettings().createCustom()`. Thus, any custom UI element must use it as its basis. The traversal methods of `CustomPanelAPI` additionally call the same methods of the plugin it is provided with (see section [`CustomPanelAPI`](#custompanelapi)). Thus, as it turns out, the most convenient way to manage both the plugin, where the logic lives, and the custom panel, where the UI hierarchy lives, is by letting the plugin own the custom panel, and let the plugin have methods that delegate the calls to the encapsulated custom panel instance. This makes the creation of more complex UI elements, be it reusable or particular, easier, as simply extending the wrapper plugin is enough to benefit from all the features of `CustomPanelAPI` without a hassle. Here is one such wrapper panel I use for my mods.

<details>
<summary>Code snippet</summary>

```java
import rolflectionlib.util.RolfLectionUtil;

public abstract class CustomPanel implements CustomUIPanelPlugin {
    private static final Object clearChildrenMethod;
    private static final Object getChildrenCopyMethod;
    private static final Object getChildrenNonCopyMethod;
    private static final Object getFaderMethod;
    private static final Object addToPositionMethod;
    private static final Object removeFromPositionMethod;
    private static final Object positionSetParentMethod;

    static {
        final UIPanelAPI panelIns = Global.getSettings().createCustom(0f, 0f, null);
        final Class<?> panelClazz = panelIns.getClass();
        final Class<?> posClazz = panelIns.getPosition().getClass();

        clearChildrenMethod = RolfLectionUtil.getMethod(
            "clearChildren", panelClazz);
        getChildrenCopyMethod = RolfLectionUtil.getMethod(
            "getChildrenCopy", panelClazz);
        getChildrenNonCopyMethod = RolfLectionUtil.getMethod(
            "getChildrenNonCopy", panelClazz);
        getFaderMethod = RolfLectionUtil.getMethod("getFader", panelClazz);
        addToPositionMethod = RolfLectionUtil.getMethod("add", posClazz, 1);
        removeFromPositionMethod = RolfLectionUtil.getMethod("remove", posClazz, 1);
        positionSetParentMethod = RolfLectionUtil.getMethod("setParent", posClazz, 1);
    }

    protected final CustomPanelAPI mPanel;

    public CustomPanel() {
        this(0f, 0f);
    }

    public CustomPanel(float width, float height) {
        mPanel = Global.getSettings().createCustom(width, height, this);
    }

    public void buttonPressed(Object buttonID) {}
    public void positionChanged(PositionAPI position) {}

    public CustomPanelAPI getPanel() { return mPanel; }
    public CustomPanelAPI panel() { return mPanel; }

    public PositionAPI getPos() { return mPanel.getPosition(); }
    public PositionAPI pos() { return mPanel.getPosition(); }

    public PositionAPI add(LabelAPI a) {
        return add((UIComponentAPI) a);
    }

    public TooltipMakerAPI getTooltip(float width, float height, boolean withScroller) {
        return mPanel.createUIElement(width, height, withScroller);
    }

    /** Note: the tooltip must be created using {@link #getTooltip}. */
    public PositionAPI add(TooltipMakerAPI a) {
        return mPanel.addUIElement(a);
    }

    public PositionAPI add(UIComponentAPI a) {
        mPanel.addComponent(a);

        return a.getPosition();
    }

    public PositionAPI add(CustomPanel a) {
        mPanel.addComponent(a.getPanel());

        return a.getPos();
    }

    public void remove(LabelAPI a) {
        remove((UIComponentAPI) a);
    }

    public void remove(UIComponentAPI a) {
        mPanel.removeComponent(a);
    }

    public void remove(CustomPanel a) {
        mPanel.removeComponent(a.getPanel());
    }

    public final Fader getPanelFader() {
        return getPanelFader(mPanel);
    }

    public PositionAPI addPositionOnly(UIComponentAPI comp) {
        return addPositionOnly(pos(), comp);
    }

    public PositionAPI removePositionOnly(UIComponentAPI comp) {
        return removePositionOnly(pos(), comp);
    }

    public void clearChildren() {
        clearChildren(mPanel);
    }

    public List<UIComponentAPI> getChildrenNonCopy() {
        return getChildrenNonCopy(mPanel);
    }

    public List<UIComponentAPI> getChildrenCopy() {
        return getChildrenCopy(mPanel);
    }

    public final void setSize(int width, int height) {
        pos().setSize(width, height);
    }

    public final void setWidth(int width) {
        pos().setSize(width, pos().getHeight());
    }

    public final void setHeight(int height) {
        pos().setSize(pos().getWidth(), height);
    }

    /** Forces a recompute of the position hierarchy by setting the size to its current value. */
    public final void posRecompute() {
        pos().setSize(pos().getWidth(), pos().getHeight());
    }

    @SuppressWarnings("unchecked")
    public static final List<UIComponentAPI> clearChildren(UIPanelAPI panel) {
        return (List<UIComponentAPI>) RolfLectionUtil.invokeMethodDirectly(clearChildrenMethod, panel);
    }

    @SuppressWarnings("unchecked")
    public static final List<UIComponentAPI> getChildrenNonCopy(UIPanelAPI panel) {
        return (List<UIComponentAPI>) RolfLectionUtil.invokeMethodDirectly(getChildrenNonCopyMethod, panel);
    }

    @SuppressWarnings("unchecked")
    public static final List<UIComponentAPI> getChildrenCopy(UIPanelAPI panel) {
        return (List<UIComponentAPI>) RolfLectionUtil.invokeMethodDirectly(getChildrenCopyMethod, panel);
    }

    public static final Fader getPanelFader(UIPanelAPI panel) {
        return (Fader) RolfLectionUtil.invokeMethodDirectly(getFaderMethod, panel);
    }

    public static final PositionAPI addPositionOnly(PositionAPI parent, UIComponentAPI comp) {
        final PositionAPI position = comp.getPosition();
        RolfLectionUtil.invokeMethodDirectly(positionSetParentMethod, position, parent);
        RolfLectionUtil.invokeMethodDirectly(addToPositionMethod, parent, position);
        return position;
    }

    public static final PositionAPI removePositionOnly(PositionAPI parent, UIComponentAPI comp) {
        final PositionAPI position = comp.getPosition();
        RolfLectionUtil.invokeMethodDirectly(positionSetParentMethod, position, (Object)null);
        RolfLectionUtil.invokeMethodDirectly(removeFromPositionMethod, parent, position);
        return position;
    }
}
```
</details>

This wrapper, which ought to get extended by all other UI elements, is minimal, as it has no instance field other than the custom panel instance it wraps. It offers many advantages over directly working with a raw `CustomPanelAPI`:
- **Encapsulation** - The panel and its logic live in one place, making the UI easier to reason about and modify.
- **Reusability** - A `CustomPanel` subclass can be instantiated and added to any parent panel.
- **Convenience** - Overloaded `add` methods reduce boilerplate for common types like `LabelAPI` and `TooltipMakerAPI`.
- **Reflection** - Common reflection calls are cached and made available with static and instance methods.

The vanilla API has a few omissions. Essential methods like `clearChildren`, `getChildrenCopy`, `getChildrenNonCopy`, and `getFader` are not exposed through `UIPanelAPI`. For these, reflection is currently required. As of the next Starsector release, several of these reflection-required methods will be available through the API. Once that happens, the reflection utilities in this pattern ought to be replaced with direct API calls. The wrapper pattern remains the cleanest way to manage complex custom panels while keeping reflection isolated in a single, well-tested place.

<br><br>

## **The Debug Panel**

The first panel to be created will draw a green, transparent quad that covers its dimensions. Additionally, it will have a `LabelAPI` with simple text.

<details>
<summary>Code snippet</summary>

```java
import java.util.List;
import java.awt.Color;

import org.lwjgl.opengl.GL11;

import com.fs.starfarer.api.Global;
import com.fs.starfarer.api.input.InputEventAPI;
import com.fs.starfarer.api.ui.Fonts;
import com.fs.starfarer.api.ui.LabelAPI;
import com.fs.starfarer.api.ui.PositionAPI;

public class DebugPanel extends CustomPanel {
    
    public DebugPanel() {
        /** The panel will have a width of 100 and height of 100. */
        super(100f, 100f);

        final LabelAPI title = Global.getSettings().createLabel("Debug Panel", Fonts.INSIGNIA_LARGE);

        /** Will be added to the top left corner of the panel with a vertical and horizontal gap of 5 pixels. */
        add(title).inTL(5f, 5f);
    }

    @Override
    public void renderBelow(float alpha) {
        /** Uses the convenience method provided by CustomPanel. */
        final PositionAPI pos = pos();
        final float x = pos.getX();
        final float y = pos.getY();
        final float w = pos.getWidth();
        final float h = pos.getHeight();

        final Color green = new Color(0f, 1f, 0f, alpha * 0.3f);

        /** Standard quad draw using immediate mode. */
        GL11.glEnable(GL11.GL_BLEND);
        GL11.glBlendFunc(GL11.GL_SRC_ALPHA, GL11.GL_ONE_MINUS_SRC_ALPHA);

        GL11.glColor4f(green.getRed() / 255f,
            green.getGreen() / 255f,
            green.getBlue() / 255f,
            green.getAlpha() / 255f);

        GL11.glBegin(GL11.GL_QUADS);
        GL11.glVertex2f(x, y);
        GL11.glVertex2f(x + w, y);
        GL11.glVertex2f(x + w, y + h);
        GL11.glVertex2f(x, y + h);
        GL11.glEnd();

        GL11.glDisable(GL11.GL_BLEND);
    }

    /** These methods are not needed for this example, but must be implemented to satisfy the interface. */
    public void render(float alpha) {}
    public void advance(float delta) {}
    public void processInput(List<InputEventAPI> events) {}
}
```
</details>

The `renderImpl` method of `CustomPanelAPI` will call the plugin method `renderBelow` before calling the render method of its children, and thus the quad will be drawn before i.e. below the children.

Now this debug panel needs to be attached somewhere. There are context-dependent panels that the API provides as a potencial parent (for the UI injection), but we want the panel to be displayed above everything else inside `TitleScreenState`, therefore we will call this method using a `BaseEveryFrameCombatPlugin`, which runs when the `CombatEngine` is used, which is the case when `TitleScreenState` is active, as the ships in the background are controlled using it. If a panel needs to be injected multiple times, i.e. the parent panel clears its children, then the `advance` method is more fit to inject the button, as it can constantly check for the presence of the injected panel, and inject it if missing.

<details>
<summary>Code snippet</summary>

```java
import com.fs.starfarer.api.GameState;
import com.fs.starfarer.api.Global;

import com.fs.starfarer.api.combat.BaseEveryFrameCombatPlugin;
import com.fs.starfarer.api.combat.CombatEngineAPI;
import com.fs.starfarer.api.ui.UIPanelAPI;
import com.fs.starfarer.title.TitleScreenState;
import com.fs.state.AppDriver;

import rolflectionlib.util.RolfLectionUtil;

public class TitleDebugScreenPlugin extends BaseEveryFrameCombatPlugin {

    public void init(CombatEngineAPI engine) {
        /** Only works when the game state is TITLE. */
        if (Global.getSettings().getCurrentState() != GameState.TITLE) return;

        /** The getScreenPanel uses the obfuscated UIPanel type, and thus must be called using reflection. */
        final Object getTitleScreenPanelMethod = RolfLectionUtil.getMethod(
            "getScreenPanel", TitleScreenState.class
        );

        final AppState title = AppDriver.getInstance().getCurrentState();
        final UIPanelAPI screen = (UIPanelAPI) RolfLectionUtil.invokeMethodDirectly(getTitleScreenPanelMethod, title);
        
        final DebugPanel debug = new DebugPanel();

        /** The getPanel call is needed, since DebugPanel is the wrapper around the actual panel, and the plugin of that panel. */
        screen.addComponent(debug.getPanel()).inTL(50f, 50f);
    }
}
```
</details>

The plugin to inject the debug panel can be registered at `settings.json`. To do this, create a `settings.json` at `data/config/settings.json` and inside it, place the following block:
```json
"plugins": {
    "TitleDebug": "path.to.plugin.TitleDebugScreenPlugin"
}
```

<br><br>

## **The Count Panel**

Now for a more involved panel (technically panel wrapper/plugin), where an internal count field is incremented when a button, which is created using a tooltip, is clicked. This updates the text of a label. To react to button clicks, an `ActionListenerDelegate` is used, which gets called by the button internally, who uses `processInputImpl` internally to check for click events (see section [ButtonAPI](#buttonapi)). When the label text is updated, its width is also updated to match the new text width, otherwise the text might wrap around to the next line.

<details>
<summary>Code snippet</summary>

```java
import java.util.List;

import com.fs.starfarer.api.Global;
import com.fs.starfarer.api.input.InputEventAPI;
import com.fs.starfarer.api.ui.Alignment;
import com.fs.starfarer.api.ui.ButtonAPI;
import com.fs.starfarer.api.ui.CutStyle;
import com.fs.starfarer.api.ui.LabelAPI;
import com.fs.starfarer.api.ui.TooltipMakerAPI;
import com.fs.starfarer.api.ui.TooltipMakerAPI.ActionListenerDelegate;
import com.fs.starfarer.api.util.Misc;

public class CountPanel extends CustomPanel {
    /** Unique Id object */
    private static final Object COUNT_BTN = new Object();

    private int count = 0;
    
    public CountPanel() {
        super(200f, 80f);

        /** Internally calls mPanel.createUIElement(width, height, withScroller); */
        final TooltipMakerAPI content = getTooltip(200f, 80f, false);

        content.addTitle("Counter", Misc.getBasePlayerColor());

        final LabelAPI counterLabel = content.addPara(getCountStr(), 5f);

        content.setButtonFontOrbitron20();
        content.setActionListenerDelegate(new ActionListenerDelegate() {
            @Override
            public void actionPerformed(Object data, Object src) {

                /** Used to identify which button was clicked. Useful for multiple buttons that share the same listener. */
                if (src instanceof ButtonAPI btn && btn.getCustomData() == COUNT_BTN) {
                    count++;
                    final String txt = getCountStr();
                    counterLabel.setText(txt);
                    /** If the width of the new text is higher, the text will be wrapped around to the next line. So resize. */
                    counterLabel.autoSizeToWidth(counterLabel.computeTextWidth(txt));
                }
            }
        });

        /** The second param is the custom data. */
        content.addButton("Increment", COUNT_BTN, Misc.getButtonTextColor(), Global.getSettings().getColor("buttonBgDark"),
            Alignment.MID, CutStyle.ALL, 70f, 28f, 10f);

        /** The addButton method adds the Button to the tooltip using addCustom already. */
        add(content);
    }

    private final String getCountStr() {
        return "Clicks: " + Integer.toString(count);
    }

    @Override public void renderBelow(float alpha) {}
    @Override public void render(float alpha) {}
    @Override public void processInput(List<InputEventAPI> events) {}
    @Override public void advance(float delta) {}
}
```
</details>

<br><br>

## **The Data List**

A very common pattern in **GUI**s are lists, in particular, scrollable ones. The `ScrollPanelAPI` implementation is obfuscated, and the interface itself is quite limited (but Alex did promise to make the API complete and add a factory method, so there is hope). Therefore the scroll panel of the tooltip will be used to hold the rows of the list. The rows will hold a faction crest and its name. Clicking any will call back a consumer, which is basically a listener without explicitly using `ActionListenerDelegate`. It could be used, but not needed in this example, so eh. The row positioning is tracked manually (out of habit) to make sure the rows are placed correctly. Notice the usage of static constants wherever fitting, which is a good habit, because it makes manual layouting (which you will need to do outside of the tooltip) much more readable. Instead of calculating the layout using number literals, the math is done using named constants, which improves readability without any runtime cost.

<details>
<summary>Code snippet</summary>

```java
import java.util.ArrayList;
import java.util.List;
import java.util.function.Consumer;

import com.fs.starfarer.api.Global;
import com.fs.starfarer.api.campaign.FactionSpecAPI;
import com.fs.starfarer.api.graphics.SpriteAPI;
import com.fs.starfarer.api.input.InputEventAPI;
import com.fs.starfarer.api.ui.Fonts;
import com.fs.starfarer.api.ui.LabelAPI;
import com.fs.starfarer.api.ui.TooltipMakerAPI;

public class FactionList extends CustomPanel {
    private static final float ROW_HEIGHT = 32f;
    private final Consumer<FactionSpecAPI> onFactionSelected;

    public FactionList(float width, float height, Consumer<FactionSpecAPI> onFactionSelected) {
        super(width, height);
        this.onFactionSelected = onFactionSelected;
        buildUI();
    }

    /** The separate method allows the UI to be rebuilt without creating a new FactionList panel. */
    public void buildUI() {
        clearChildren();

        final List<FactionSpecAPI> factions = new ArrayList<>();
        for (FactionSpecAPI faction : Global.getSettings().getAllFactionSpecs()) {
            if (faction.isShowInIntelTab()) {
                factions.add(faction);
            }
        }

        /** Create a scrollable tooltip as the content container */
        final TooltipMakerAPI container = getTooltip(pos().getWidth(), pos().getHeight(), true);

        float yCoord = 0f;
        for (FactionSpecAPI faction : factions) {
            final FactionRow row = new FactionRow(
                pos().getWidth(), ROW_HEIGHT,
                faction, onFactionSelected
            );
            /** Manually positioned instead of trusting the relative positioning of addCustom. */
            container.addCustom(row.getPanel(), 0f).getPosition().inTL(0f, yCoord);
            yCoord += ROW_HEIGHT + 5f;
        }

        /** Needed, since the manual positioning can mess with the height tracking of the tooltip. */
        container.setHeightSoFar(yCoord);
        add(container);
    }

    @Override public void renderBelow(float alpha) {}
    @Override public void render(float alpha) {}
    @Override public void advance(float delta) {}
    @Override public void processInput(List<InputEventAPI> events) {}

    private static class FactionRow extends CustomPanel {
        private final FactionSpecAPI faction;
        private final Consumer<FactionSpecAPI> onSelect;
        private final SpriteAPI crest;
        private final int ICON_S = 28;

        public FactionRow(float width, float height, FactionSpecAPI faction, Consumer<FactionSpecAPI> onSelect) {
            super(width, height);
            this.faction = faction;
            this.onSelect = onSelect;
            /** This is fine, the underlying id map can handle null keys, should the faction crest be null for some reason. */
            this.crest = Global.getSettings().getSprite(faction.getCrest());
            if (crest != null) crest.setSize(ICON_S, ICON_S);

            final LabelAPI nameLabel = Global.getSettings().createLabel(
                faction.getDisplayName(),
                Fonts.ORBITRON_12
            );
            nameLabel.setColor(faction.getBaseUIColor());
            add(nameLabel).inLMid(ICON_S + 8f);
        }

        @Override
        public void processInput(List<InputEventAPI> events) {
            for (InputEventAPI event : events) {
                if (event.isConsumed()) continue;

                if (event.isLMBEvent() && pos().containsEvent(event)) {
                    event.consume();
                    onSelect.accept(faction);
                    return;
                }
            }
        }

        @Override
        public void render(float alpha) {
            if (crest != null) {
                /** Basically vertical mid. */
                final float y = (pos().getHeight() - ICON_S) / 2f;
                /** Should not touch the left border, or might look ugly. */
                crest.render(3f, y);
            }
        }

        @Override public void renderBelow(float alpha) {}
        @Override public void advance(float delta) {}
    }
}
```
</details>

<br><br>

## **The Detail Panel**

Another common pattern in building UI is creating an element that builds UI for a specific instance of data. Rows that display the details of a faction are one example (see section [The Data List](#the-data-list)). But another one is a detail panel, that gets updated when the user selects from a list of data. Continuing from the last example, where a `FactionSpecAPI` is selected from a list, this selection then invokes a consumer. This consumer can be used to build a detail panel, which displays more information in the context of that faction. For simplicity, the example below displays the name, logo, and the first description paragraph of the faction, similar to the **factions** tab inside the **Intel** tab. The panel implements `Consumer<FactionSpecAPI>`, and is passed to `FactionList` as its consumer.

<details>
<summary>Code snippet</summary>

```java
import java.util.List;
import java.util.function.Consumer;

import com.fs.starfarer.api.Global;
import com.fs.starfarer.api.campaign.FactionSpecAPI;
import com.fs.starfarer.api.input.InputEventAPI;
import com.fs.starfarer.api.loading.Description;
import com.fs.starfarer.api.loading.Description.Type;
import com.fs.starfarer.api.ui.Alignment;
import com.fs.starfarer.api.ui.Fonts;
import com.fs.starfarer.api.ui.LabelAPI;
import com.fs.starfarer.api.ui.TooltipMakerAPI;

public class FactionDetailPanel extends CustomPanel implements Consumer<FactionSpecAPI> {
    private FactionSpecAPI currentFaction;

    public FactionDetailPanel(float width, float height) {
        super(width, height);

        showDefaultState();
    }

    private void showDefaultState() {
        clearChildren();

        /** Please avoid using the tooltip for a singular label. The tool is overqualified for the job. */
        final LabelAPI defaultLbl = Global.getSettings().createLabel("Select a faction from the list.", Fonts.DEFAULT_SMALL);
        /** Should stand in the middle. */
        add(defaultLbl).inMid();
    }

    @Override
    public void accept(FactionSpecAPI faction) {
        currentFaction = faction;
        buildUI();
    }

    private void buildUI() {
        clearChildren();
        if (currentFaction == null) {
            showDefaultState();
            return;
        }

        /** Vanilla uses these values quite often. I also do for visual consistency. */
        final float pad = 3f;
        final float opad = 10f;

        final float width = pos().getWidth();
        final float height = pos().getHeight();

        /** It is sort of justified here, because it provides a sprite wrapper using addImage. */
        final TooltipMakerAPI content = getTooltip(width, height, false);

        final String logoId = currentFaction.getLogo();
        if (logoId != null) {
            /** The logo is usually 410*256 px, so scale the width/height by that. */
            content.addImage(logoId, 200f, 125f, pad);
        }

        /** Do not use addTitle, as that would place the title at the top left, which is where the logo sits. */
        final LabelAPI nameLabel = content.addPara(currentFaction.getDisplayName(), currentFaction.getBaseUIColor(), pad);
        /** Remember that the width of the label is the width of the tooltip, so centering it at the middle works. */
        nameLabel.setAlignment(Alignment.MID);
        /** A section heading could have also been used. Personal taste. */

        /** Let me be honest, I have no idea if this is the correct way to retrieve the faction desc, but this is a GUI guide, so don't care. */
        final Description desc = Global.getSettings().getDescription(currentFaction.getId(), Type.FACTION);
        content.addPara(desc.getText1(), opad);

        /** Default is inBL(0f, 0f), which is fine, because the tooltip covers the entire panel. */
        add(content);
    }

    @Override public void renderBelow(float alpha) {}
    @Override public void render(float alpha) {}
    @Override public void advance(float delta) {}
    @Override public void processInput(List<InputEventAPI> events) {}
}
```
</details>