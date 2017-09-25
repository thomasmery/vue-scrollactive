<template>
    <nav class="scrollactive-nav">
        <slot></slot>
    </nav>
</template>

<script>
import bezierEasing from 'bezier-easing';

export default {
    props: {
        /**
        * Class that will be applied in the menu item.
        *
        * @default  'is-active'
        * @type {String}
        */
        activeClass: {
            type: String,
            default: 'is-active',
        },

        /**
        * Amount of space between top of screen and the section to
        * highlight. (Usually your fixed header's height)
        *
        * @default 20
        * @type {Number}
        */
        offset: {
            type: Number,
            default: 20,
        },

        /**
        * Enables/disables the scrolling when clicking in a menu item.
        * Disable if you'd like to handle the scrolling by your own.
        *
        * @default true
        * @type {Boolean}
        */
        clickToScroll: {
            type: Boolean,
            default: true,
        },

        /**
        * The duration of the scroll animation when clicking to scroll
        * is activated.
        *
        * @default 600
        * @type {Number}
        */
        duration: {
            type: Number,
            default: 600,
        },

        /**
        * Defines if the plugin should track the section change when
        * clicking an item to scroll to its section. If set to true,
        * it will always keep track and change the active class to the
        * current section while scrolling, if false, the active class
        * will be immediately applied to the clicked menu item, ignoring
        * the passed sections until the scrolling is over.
        *
        * @default false
        * @type {Boolean}
        */
        alwaysTrack: {
            type: Boolean,
            default: false,
        },

        /**
        * Your custom easing value for the click to scroll functionality.
        * It must be a string with 4 values separated by commas in a
        * cubic bezier format.
        *
        * @default '.5,0,.35,1'
        * @type {String}
        */
        bezierEasingValue: {
            type: String,
            default: '.5,0,.35,1',
        },
    },

    data() {
        return {
            scrollactiveItems: null,
            bezierEasing,
            lastActiveItem: null,
        };
    },

    computed: {
        /**
        * Transforms the bezier easing string value into an array.
        *
        * @return {Array}
        */
        cubicBezierArray() {
            return this.bezierEasingValue.split(',');
        },


    },

    methods: {

        getCurrentItem() {
            const distanceFromTop = window.scrollY;
            let currentItem = null;

            this.scrollactiveItems.forEach((scrollactiveItem) => {

                const targetId = scrollactiveItem.hash.substr(1);
                const target = document.getElementById(targetId);

                scrollactiveItem.classList.remove(this.activeClass);

                let targetIsInView = false;
                let targetInViewTop = this.getOffsetTop(target) - this.offset;
                let targetInViewBottom = targetInViewTop + target.clientHeight;

                targetIsInView = distanceFromTop >= targetInViewTop
                    && distanceFromTop < targetInViewBottom;

                if (targetIsInView) {
                    currentItem = scrollactiveItem;
                }

            });

            return currentItem;

        },
        /**
        * Will be called when scrolling event is triggered to handle
        * the addition of the active class in the current section item
        * and fire the change event.
        */
        onScroll(event) {
            let currentItem = this.getCurrentItem();
            console.log('scroll');
            if (currentItem !== this.lastActiveItem) {
                this.$emit('itemchanged', event, currentItem, this.lastActiveItem);
                this.lastActiveItem = currentItem;
            }

            if (currentItem) currentItem.classList.add(this.activeClass);
        },

        /**
        * Sets the initial list of menu items, validating if its hash
        * corresponds to a valid element ID.
        */
        setScrollactiveItems() {
            const scrollactiveItems = document.querySelectorAll('.scrollactive-item');

            scrollactiveItems.forEach((scrollactiveItem) => {
                const targetId = scrollactiveItem.hash.substr(1);
                const target = document.getElementById(targetId);
                if (!target) {
                    throw new Error(`Element '${scrollactiveItem.hash}' was not found. Make sure it is set in the DOM.`);
                }
            });

            this.scrollactiveItems = scrollactiveItems;

            if (this.clickToScroll) {
                scrollactiveItems.forEach((scrollactiveItem) => {
                    scrollactiveItem.addEventListener('click', this.itemClickHandler);
                });
            } else {
                scrollactiveItems.forEach((scrollactiveItem) => {
                    scrollactiveItem.removeEventListener('click', this.itemClickHandler);
                });
            }
        },

        /**
        * Triggers the scrolling when clicking a menu item.
        */
        itemClickHandler(event) {
            event.preventDefault();

            const item = event.currentTarget;
            const targetId = item.hash.substr(1);
            this.scrollToTarget(targetId);

            // notify listeners that we change section
            this.$emit('itemchanged', event, item);
        },


        scrollToTarget(targetId) {
            const vm = this;

            if(typeof targetId  !== 'string') {
                throw new Error(`scrollToTarget method requires a DOM Id as parameter.`);
            }

            const target = document.getElementById(targetId);

            // we just want the animation to the target
            if (!this.alwaysTrack) {
                window.removeEventListener('scroll', this.onScroll);
                window.cancelAnimationFrame(window.AFRequestID);
            }

            const targetDistanceFromTop = this.getOffsetTop(target);
            const startingY = window.pageYOffset;
            const difference = targetDistanceFromTop - startingY;
            const easing = vm.bezierEasing(
                this.cubicBezierArray[0],
                this.cubicBezierArray[1],
                this.cubicBezierArray[2],
                this.cubicBezierArray[3],
            );
            let start = null;

            function step(timestamp) {
                if (!start) start = timestamp;

                let progress = timestamp - start;
                let progressPercentage = progress / vm.duration;

                if (progress >= vm.duration) progress = vm.duration;
                if (progressPercentage >= 1) progressPercentage = 1;

                const perTick = startingY + (easing(progressPercentage) * (difference - vm.offset));

                window.scrollTo(0, perTick);

                if (progress < vm.duration) {
                    window.AFRequestID = window.requestAnimationFrame(step);
                } else {
                    if (!vm.alwaysTrack) {
                        const currentItem = vm.getCurrentItem();
                        if (currentItem) currentItem.classList.add(vm.activeClass);

                        // prevent triggering a scroll event at the end of the scroll animation
                        // if we have removed the on scroll handler we don't want a 'ghost' event to be triggered
                        // there is probably a better way than setTimeut though
                        setTimeout(function() {
                            window.addEventListener('scroll', vm.onScroll);
                        }, 100);

                    }
                }
            }

            window.requestAnimationFrame(step);
        },

        /**
        * Gets the top offset position of an element in the document.
        *
        * @param  {Element} element
        * @return {Number}
        */
        getOffsetTop(element) {
            let yPosition = 0;
            let nextElement = element;

            while (nextElement) {
                yPosition += (nextElement.offsetTop);
                nextElement = nextElement.offsetParent;
            }

            return yPosition;
        },
    },

    mounted() {
        this.setScrollactiveItems();
        this.onScroll();
        window.addEventListener('scroll', this.onScroll);
    },

    updated() {
        this.setScrollactiveItems();
    },

    beforeDestroy() {
        window.removeEventListener('scroll', this.onScroll);
        window.cancelAnimationFrame(window.AFRequestID);
    },
};
</script>
