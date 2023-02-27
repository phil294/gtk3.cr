# GTK3 Crystal Bindings

This is a fork of [hugopl/gtk4.cr](https://github.com/hugopl/gtk4.cr), but with GTK3. Please see that other repo for details.

There's also the more popular and more battle-tested [jhass/crystal-gobject](https://github.com/jhass/crystal-gobject) for GTK3. Both bindings are not compatible, but migrating is not hard. I made this very alternative because when I used the other one, there were rare different rare crashes which I couldn't resolve.

If you need AT-SPI, you can use https://github.com/phil294/atspi.cr.

## GC-related problems

Crystal's garbage collector [GC] normally runs in somewhat unpredictable intervals. This can lead to crashes, sometimes. The following precautions seem to resolve them (entirely?). Maybe they only occur with `-Dpreview_mt` only anyway.

You'll see a lot of `GLib-GObject-CRITICAL **:` etc. error messages in the console all the time. I guess they're not avoidable for now.

1. Call GC manually before and after running Gtk code. Example: Instead of
    ```crystal
    GLib.idle_add do { gtk code... }
    ```
    do
    ```crystal
    GC.collect
    GLib.idle_add do { gtk code...}
    GC.collect
    ```
    Those Glib error messages mentioned above will still show up, but now exactly when you instruct it to, so it doesn't interfere with the Gtk thread.
1. Avoid `begin`/`rescue` inside destroy handlers. Example:
    ```crystal
    window.destroy_signal.connect do
      begin
        raise "asdf"
      rescue e
        STDERR.puts e
    end
    ```
    this will somehow lead to crashes
1. When you have many invocations of `idle_add` in a short timeframe (e.g. 1000 within 1 second), you should wait for each of those to complete before querying the next to avoid crashes. Example: Instead of
    ```crystal
    def my_gtk_act_function(&block : -> nil)
        GC.collect
        GLib.idle_add do
            block.call
            false
        end
        GC.collect
    end
    1000.times { my_gtk_act_function { ... } }
    ```
    do
    ```crystal
    def my_gtk_act_function(&block : -> nil)
        channel = Channel(Nil).new
        GC.collect
        GLib.idle_add do
            block.call
            channel.send(nil)
            false
        end
        channel.receive
        GC.collect
    end
    1000.times { my_gtk_act_function { ... } }
    ```
    Where the second method also offers the capability of error handling or returning values as well, at the expense of speed.
