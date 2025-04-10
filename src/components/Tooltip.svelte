<script>
    export let text = '';
    export let position = 'bottom';
</script>

<div class="tooltip-container">
    <slot></slot>
    <div class="tooltip" class:bottom={position === 'bottom'}>
        {text}
    </div>
</div>

<style>
    .tooltip-container {
        position: relative;
        display: inline-block;
        font-family: 'Helvetica Neue', sans-serif;
    }

    .tooltip {
        visibility: hidden;
        position: absolute;
        background-color: white;
        color: #374151;
        text-align: left;
        padding: 0.75rem 1rem;
        font-size: 0.875rem;
        width: max-content;
        max-width: 300px;
        border: 1px solid #43485A;
        box-shadow: 4px 4px 0 0 #d6613b;
        opacity: 0;
        transition: opacity 0.2s ease;
        pointer-events: none;
        font-weight: 500;
        z-index: 2000;
        font-family: 'Helvetica Neue', sans-serif;
    }

    .tooltip.bottom {
        top: 100%;
        left: 50%;
        transform: translateX(-50%);
        margin-top: 10px;
    }

    .tooltip-container:hover .tooltip {
        visibility: visible;
        opacity: 1;
    }

    @media (max-width: 768px) {
        .tooltip {
            font-size: 0.7rem;
            max-width: 150px;
        }
    }
</style>