# Query Monitor - MODIFIED

## Modifications
* Query Monitor displays memory and time usage for ever callback on the Hooks & Actions panel.
* Empty hooks and non-triggered callbacks are not displayed.
    * So `define( 'QM_SHOW_ALL_HOOKS', true )` can be used to activate the display of filters without displaying non-triggered stuff.


## Requirements
* Modification of WordPress core files (see Installation step 3).
* PHP 8.2

    > PHP 8.2 introduced `memory_reset_peak_usage()`, which is used here to get the peak memory usage of ever callback.

## Installation
1. Download to `wp-content/plugins/query-monitor` (name is important).
2. Run `composer install; npm install; npm build`
3. WordPress Core needs to be adjusted to collect the data.

    1. Open `class-wp-hook.php`
    2. Find around line 315
        ```php
        foreach ( $this->callbacks[ $priority ] as $the_ ) {
            if ( ! $this->doing_action ) {
                $args[0] = $value;
            }

            // Avoid the array_slice() if possible.
            if ( 0 === $the_['accepted_args'] ) {
                $value = call_user_func( $the_['function'] );
            } elseif ( $the_['accepted_args'] >= $num_args ) {
                $value = call_user_func_array( $the_['function'], $args );
            } else {
                $value = call_user_func_array( $the_['function'], array_slice( $args, 0, $the_['accepted_args'] ) );
            }
        }
        ```
    3. Replace with
        ```php
        // MODIFICATION ON NEXT LINE: Changed $the_ to &$the_.
        foreach ( $this->callbacks[ $priority ] as &$the_ ) {
            if ( ! $this->doing_action ) {
                $args[0] = $value;
            }

            //- MODIFICATION START
            $before_memory = memory_get_usage();
            // Reset memory peak usage stats.
            memory_reset_peak_usage();
            $time_start = microtime(true);
            //- MODIFICATION END

            // Avoid the array_slice() if possible.
            if ( 0 === $the_['accepted_args'] ) {
                $value = call_user_func( $the_['function'] );
            } elseif ( $the_['accepted_args'] >= $num_args ) {
                $value = call_user_func_array( $the_['function'], $args );
            } else {
                $value = call_user_func_array( $the_['function'], array_slice( $args, 0, $the_['accepted_args'] ) );
            }

            //- MODIFICATION START
            $the_['time_used'] = round(microtime(true) - $time_start, 4) . ' s';
            $the_['memory_added'] = round((memory_get_usage()-$before_memory) / 1024) . ' KB';
            $the_['peak_memory_usage'] = round((memory_get_peak_usage()-$before_memory) / 1024) . ' KB';
            //- MODIFICATION END
		}
        ```
