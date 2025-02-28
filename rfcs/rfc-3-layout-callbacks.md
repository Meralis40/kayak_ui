# RFC 3 - Layout Callbacks

## Background

In my attempts to create a `ScrollBox` widget, I found that I first needed to calculate the actual size of the content to be scrolled. My naive approach was just to get all children of the `ScrollBox`, get their layout, and calculate the total content size. However, this has a couple issues:

1. It's **fragile** - We only receive the last calculated layout and may need to force a render with `context.mark_dirty()` until we don't detect any changes.
2. It's **dumb** - Since we only force a re-render when the layout is different, layouts that take more than a frame to calculate are not caught (specifically, surrounding our content in an `Element` with width and height set to `Units::Auto` will not be calculated correctly).

Because of this, I think we need a better solution for getting layout information.

## Problem

There's currently no proper way to respond to a widget layout. We either need to just hope it's fully calculated within a set number of renders, set it manually, or perform some other hack. Instead, we need some way to properly be notified of a widget's layout.

## Solution

There's a few ways to go about this, but one of the simplest will be to add an `on_layout` event callback in manner similar to React Native's [`onLayout`](https://reactnative.dev/docs/view#onlayout) prop.

Additionally, morphorm exposes a [`GeometryChanged`](https://github.com/geom3trik/morphorm/blob/cd4032e4409e913b4b4f2f7424761530d1652970/src/types.rs#L98-L119) bitflag struct that could be passed along to the end user so they know if anything changed or *will* change.

### Desired API

The desired API would then look something like:

```rust
#[widget]
fn MyWidget() {
  let on_layout = OnLayout::new(|ctx, event| {
    // --- Get Current Layout --- //
    let width = event.layout.width;
    let height = event.layout.height;
    let x = event.layout.x;
    let y = event.layout.y;
    let z = event.layout.z;
    
    // --- Get Change Flags --- //
    let pos_change = GeometryChanged::POSX_CHANGED | GeometryChanged::POSY_CHANGED;
    if(pos_change == event.flags) {
      println!("Position changed!");
    }
    
    // --- Do Something With Widget --- //
    let widget_id = event.target;
    // ...
    
  });
  
  rsx! {
    <Element on_layout={Some(on_layout)} />
  }
}
```

### Data Types

At a basic level, this API would need the following types: `Layout`, `LayoutEvent`, and `OnLayout`.

#### `Layout`

While we already have a `Rect` struct that carries this type of data, it might be better to split away from using this struct directly in case we need to add/remove fields. So we'll create a `Layout` type instead.

```rust
pub struct Layout {
  pub width: f32,
  pub height: f32,
  pub x: f32,
  pub y: f32,
  pub z: f32,
}

impl Layout {
  /// Returns the position as a Kayak position type (which is currently just a tuple)
  pub fn pos(&self) -> (f32, f32) {/* ... */}
}

impl From<Layout> for Rect {/* ... */}
impl From<Rect> for Layout {/* ... */}
```

#### `LayoutEvent`

This type will contain the actual event data corresponding to the layout.

```rust
pub struct LayoutEvent {
  pub layout: Layout,
  pub flags: GeometryChanged,
  pub target: Index,
}
```

#### `OnLayout`

This type is identical to `OnEvent` except that it passes `LayoutEvent` instead of a plain `Event`.

### Implementation

#### Standalone Event

The `LayoutEvent` should *not* be part of the standard `Event` type. The reasoning for this is twofold:

1. The `EventType` enum specifically contains only interaction events (keyboard events, cursor events, focus events, etc.). Layout is not an interaction and would be out-of-place in such an enum.
2. We currently only support a single `on_event` handler. This may change in the future, but right now it would mean that layout handling would need to be defined alongside interaction/input handling. While this is technically okay, it may lead to messier code due to the inability to separate concerns as effectively.

However, this means we need a new API on widgets that allows for this new event.

#### Widgets and Props

Like with `OnEvent`, we'll need to store this prop type on the widget somehow. This means we'll need a new method to retrieve the event type and a new derive macro helper parameter to auto-generate this method.

```rust
trait WidgetProps: /* ... */ {
  // ...
  fn get_on_layout(&self) -> Option<OnLayout>;
}
```

Which could be generated by the derive helper macro:

```rust
#[derive(WidgetProps, Default, Debug, Clone, PartialEq)]
struct MyWidgetProps {
  #[prop_field(OnLayout)]
  on_layout: Option<OnLayout>
}
```

And easily accessed by widget via;

```rust
trait WidgetProps: /* ... */ {
  // ...
  fn on_layout(&mut self, context: &mut KayakContextRef, event: &mut LayoutEvent);
}
```

##### Considerations

Of course, it's important to consider that this makes the `on_layout` prop completely optional. If a third party widget crate created, they will need to add this field to their widgets in order for users to access the event.

An alternative to this would be to put `on_layout` on `Widget` directly. However, this becomes messy because we'd need to parse that prop distinctly from the others. Additionally, layout events are worthless on widgets that don't have a layout (i.e., they're `RenderCommand::Empty`).

For these reasons, it makes the most sense to put it on `WidgetProps` and let the user opt-in to them, despite it possibly being a bit annoying for library authors.

#### Dispatching the Event

##### Management

The big question for this feature is *who's in charge of the event?* Obviously, it's an internal thing that users and integration crates won't need to manage themselves. But should the responsibility fall on `WidgetManager` or `KayakContext`? And should it go through `EventDispatcher`, a custom dispatcher, or just a simple function?

While it would make sense to store this on `WidgetManager`, it's not possible due to the requirement that we pass a mutable `KayakContext` to the layout event. This means the dispatching must take place within `KayakContext`. 

As for which dispatcher to use, it might be worth creating a custom `LayoutEventDispatcher` to handle the layout events. The reason for this is purely organizational. It's probably better to separate out the concerns so that `KayakContext` doesn't get too bloated.

##### Logic

One complication with the logic is determining which widgets to notify. ~~To keep it simple, we can just notify all widgets that have been re-rendered.~~ To be consistent with similar frameworks (specifically React Native), we'll only notify widgets whose layout has *changed*.

Another additional complication is that we can't just dispatch the event over all indices in `WidgetManager::dirty_nodes` since those only contain widgets that requested a re-render (i.e., their state updated or a binding marked them dirty). Iterating over only those would skip out on all the children, which may have had their own layouts changed after rendering.

This means we actually need to use `WidgetManager::dirty_render_nodes` in order to get all re-rendered nodes. So instead of draining this collection in `WidgetManager::render()`, we'll need to keep them around and drain them after calculating the layout in `KayakContext::render()`— which we'll do in `LayoutEventDispatcher::dispatch(...)`.

Lastly, we need to notify the parent of the affected subtree that their children have had their layout changed in some way. To do this, we'll add the widget's parent to an `IndexSet` that will be processed after all the dirty nodes. In doing so, we will need to consider two things: (1) there may be more than one sub-tree in being rendered and (2) parents only care if their immediate children change.

The first consideration is especially important when dealing with parents because it affects *how* we process them. Since there can be more than one sub-tree (two unrelated parts of the entire widget tree), we can't optimize by only getting the parent of the last dirty node. Though it would be faster, this might skip a sub-tree!

##### `LayoutEventDispatcher`

The dispatcher will look something like this:

```rust
pub(crate) struct LayoutEventDispatcher;

impl LayoutEventDispatcher {
  
  pub fn dispatch(context: &mut KayakContext) {
    
    let dirty_nodes: Vec<Index> = context.widget_manager.dirty_render_nodes
      .lock()
      .expect("Couldn't get lock on dirty render nodes!")
      .drain(..)
      .collect();
    
    // Use IndexSet to prevent duplicates and maintain speed
    let mut parents: IndexSet<Index> = IndexSet::default();
    
    for node_index in dirty_nodes) {
      // If layout is not changed -> skip
      
      // Add parent to set
      let parent_index = /* ... */
      parents.insert(parent_index);
      
      // Process and dispatch
      Self::process(node_index, context);
    }
  
  	// Finally, process all parents
    for parent_index in parents) {
      // Process and dispatch
      Self::process(parent_index, context);
    }
  }

	fn process(index: Index, context: &mut KayakContext) {
    // We should be able to just get layout from WidgetManager here
    // since the layouts will be calculated by this point
    let mut layout_event = /* ... */

    let mut widget = context.widget_manager.take(node_index);
    let mut context_ref = KayakContextRef::new(context, Some(node_index));
    widget.on_layout(&mut context_ref, &mut layout_event);
    context.widget_manager.repossess(widget);
  }

}
```

And callable like:

```rust
impl KayakContext {
  // ...
  
  pub fn render(&mut self) {
    // ...
    
    // Render
    self.widget_manager.render();
    // Calculate layout
    self.widget_manager.calculate_layout();
    // ADD THIS:
    LayoutEventDispatcher::dispatch(&mut self);

    // ...
  }
}
```

> Note: The code snippets above are *not* exact outlines for the actual implementation. They're guides to show how this might be accomplished.

## TL;DR

1. Create `Layout`, `LayoutEvent`, and `OnLayout`
2. Add `OnLayout` prop field and method to `WidgetProps` and `Widget`
3. Create and integrate `LayoutEventDispatcher`