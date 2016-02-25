# sturdy-fiesta
+    Clutter.Actor? out_group = null;
 +    Clutter.Actor? in_group = null;
 +    public override void kill_switch_workspace()
 +    {
 +        if (this.out_group != null) {
 +            out_group.transitions_completed();
 +        }
 +    }
 +
 +    void switch_workspace_done()
 +    {
 +        var screen = this.get_screen();
 +
 +        foreach (var actor in Meta.Compositor.get_window_actors(screen)) {
 +            Clutter.Actor? orig_parent = actor.get_data("orig-parent");
 +
 +            if (orig_parent == null) {
 +                continue;
 +            }
 +
 +            actor.ref();
 +            actor.get_parent().remove_child(actor);
 +            orig_parent.add_child(actor);
 +            actor.unref();
 +
 +            actor.set_data("orig-parent", null);
 +        }
 +
 +        SignalHandler.disconnect_by_func(out_group, (void*)switch_workspace_done, this);
 +
 +        out_group.remove_all_transitions();
 +        in_group.remove_all_transitions();
 +        out_group.destroy();
 +        out_group = null;
 +        in_group.destroy();
 +        in_group = null;
 +
 +        this.switch_workspace_completed();
 +    }
 +
 +
 +    public static const int SWITCH_TIMEOUT = 250;
 +    public override void switch_workspace(int from, int to, Meta.MotionDirection direction)
 +    {
 +        int screen_width;
 +        int screen_height;
 +
 +        if (from == to) {
 +            this.switch_workspace_completed();
 +            return;
 +        }
 +
 +        out_group = new Clutter.Actor();
 +        in_group = new Clutter.Actor();
 +
 +        var screen = this.get_screen();
 +        var stage = Meta.Compositor.get_stage_for_screen(screen);
 +
 +        stage.add_child(in_group);
 +        stage.add_child(out_group);
 +        stage.set_child_above_sibling(in_group, null);
 +
 +        screen.get_size(out screen_width, out screen_height);
 +
 +        /* TODO: Windows should slide "under" the panel/dock
 +         * Move "in-between" workspaces, e.g. 1->3 shows 2 */
 +
 +
 +        foreach (var actor in Meta.Compositor.get_window_actors(screen)) {
 +            var window = actor.get_meta_window();
 +
 +            if (!window.showing_on_its_workspace() || window.is_on_all_workspaces()) {
 +                continue;
 +            }
 +
 +            var space = window.get_workspace();
 +            var win_space = space.index();
 +
 +            if (win_space == to || win_space == from) {
 +
 +                var orig_parent = actor.get_parent();
 +                unowned Clutter.Actor? new_parent = win_space == to ? in_group : out_group;
 +                actor.set_data("orig-parent", orig_parent);
 +
 +                actor.ref();
 +                orig_parent.remove_child(actor);
 +                new_parent.add_child(actor);
 +                actor.unref();
 +            }
 +        }
 +
 +        int y_dest = 0;
 +        int x_dest = 0;
 +
 +        if (direction == Meta.MotionDirection.UP ||
 +            direction == Meta.MotionDirection.UP_LEFT ||
 +            direction == Meta.MotionDirection.UP_RIGHT) {
 +            y_dest = screen_height;
 +        } else if (direction == Meta.MotionDirection.DOWN ||
 +                    direction == Meta.MotionDirection.DOWN_LEFT ||
 +                    direction == Meta.MotionDirection.DOWN_RIGHT) {
 +            y_dest = -screen_height;
 +        }
 +
 +        if (direction == Meta.MotionDirection.LEFT ||
 +            direction == Meta.MotionDirection.UP_LEFT ||
 +            direction == Meta.MotionDirection.DOWN_LEFT) {
 +            x_dest = screen_width;
 +        } else if (direction == Meta.MotionDirection.RIGHT ||
 +                    direction == Meta.MotionDirection.UP_RIGHT ||
 +                    direction == Meta.MotionDirection.DOWN_RIGHT) {
 +            x_dest = -screen_width;
 +        }
 +
 +        in_group.set_position(-x_dest, -y_dest);
 +        in_group.save_easing_state();
 +        in_group.set_easing_mode(Clutter.AnimationMode.EASE_OUT_QUAD);
 +        in_group.set_easing_duration(SWITCH_TIMEOUT);
 +        in_group.set_position(0, 0);
 +        in_group.restore_easing_state();
 +
 +        out_group.transitions_completed.connect(switch_workspace_done);;
 +
 +        out_group.save_easing_state();
 +        out_group.set_easing_mode(Clutter.AnimationMode.EASE_OUT_QUAD);
 +        out_group.set_easing_duration(SWITCH_TIMEOUT);
 +        out_group.set_position(x_dest, y_dest);
 +        out_group.restore_easing_state();
