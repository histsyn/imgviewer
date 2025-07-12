// ==UserScript==
// @name         StyleCloset Image Viewer
// @namespace    http://tampermonkey.net/
// @version      1.2
// @description  Extract and view product gallery images from styleclosetkr.com
// @author       You
// @match        https://www.styleclosetkr.com/*
// @match        https://myaleshia.com/*
// @match        https://www.doublevofficial.co/*
// @match        https://www.wwhitestudio.com/*
// @grant        none
// @require      https://cdnjs.cloudflare.com/ajax/libs/openseadragon/4.1.0/openseadragon.min.js
// ==/UserScript==

(function() {
    'use strict';

    let extractedImages = [];
    let extractedThumbs = [];
    let currentImageIndex = 0;
    let viewer = null;
    let overlayContainer = null;
    let isFullScreen = false;

    function createFloatingBox() {
        const floatingBox = document.createElement('div');
        floatingBox.id = 'image-extractor-box';
        floatingBox.style.cssText = `
            position: fixed;
            bottom: 20px;
            right: 20px;
            width: 200px;
            height: 120px;
            background: #fff;
            border: 2px solid #333;
            border-radius: 10px;
            box-shadow: 0 4px 12px rgba(0,0,0,0.3);
            z-index: 10000;
            padding: 10px;
            font-family: Arial, sans-serif;
            display: flex;
            flex-direction: column;
            gap: 8px;
        `;

        const retrieveBtn = document.createElement('button');
        retrieveBtn.textContent = 'Retrieve';
        retrieveBtn.style.cssText = `
            padding: 8px 12px;
            background: #007bff;
            color: white;
            border: none;
            border-radius: 5px;
            cursor: pointer;
            font-size: 12px;
            transition: background 0.3s;
        `;
        retrieveBtn.onmouseover = () => retrieveBtn.style.background = '#0056b3';
        retrieveBtn.onmouseout = () => retrieveBtn.style.background = '#007bff';

        const showBtn = document.createElement('button');
        showBtn.textContent = 'Show';
        showBtn.disabled = true;
        showBtn.style.cssText = `
            padding: 8px 12px;
            background: #6c757d;
            color: white;
            border: none;
            border-radius: 5px;
            cursor: not-allowed;
            font-size: 12px;
            transition: background 0.3s;
        `;

        retrieveBtn.addEventListener('click', extractImages);
        showBtn.addEventListener('click', showImages);

        floatingBox.appendChild(retrieveBtn);
        floatingBox.appendChild(showBtn);
        document.body.appendChild(floatingBox);

        return { retrieveBtn, showBtn };
    }

    function extractImages() {
        const galleryItems1 = document.querySelectorAll('.ProductGallery-item-image');
        const galleryItems2 = document.querySelectorAll('.product-media.product-media--image img.theme-img');
        const galleryItems3 = document.querySelectorAll('.ProductDetail-gallery img');

        let count1 = 0, count2 = 0, count3 = 0;

        // Extractor 1
        galleryItems1.forEach(item => {
            const style = item.getAttribute('style');
            if (style) {
                const urlMatch = style.match(/url\("([^"]+)"\)/);
                if (urlMatch && urlMatch[1]) {
                    count1++;
                    let imageUrl = urlMatch[1];
                    if (window.location.href.startsWith('https://www.wwhitestudio.com')) {
                        imageUrl = imageUrl.replace(/(\d+x\.JPG.*)/, 'original.JPG');
                    }else {
                        imageUrl = imageUrl.replace(/\d+x\.webp\?source_format=jpg/g, 'original.jpg');
                    }
                    extractedImages.push(imageUrl);

                    let imageThumbs = "";
                    if (window.location.href.startsWith('https://www.wwhitestudio.com')) {
                        imageThumbs = imageUrl.replace(/original\.JPG$/, '140x.JPG');
                    }else {
                        imageThumbs = imageUrl.replace(/original\.jpg$/, '140x.webp?source_format=jpg');
                    }
                    extractedThumbs.push(imageThumbs);
                }
            }
        });

        // Extractor 2
        galleryItems2.forEach(img => {
            let imageUrl = img.srcset ?
                img.srcset.split(',')
                        .pop()
                        .trim()
                        .split(' ')[0] :
                img.src;

            if (imageUrl) {
                count2++;
                let imageUrl2 = imageUrl.replace(/(\?v=\d+)(&|\?).*/, '');
                extractedImages.push(imageUrl2);
                let imageThumbs = imageUrl.replace(/width=\d.*/, 'width=130');
                extractedThumbs.push(imageThumbs);
            }
        });

        // Extractor 3
        galleryItems3.forEach(img => {
            const srcset = img.getAttribute('data-srcset') || img.srcset || '';
            const src = img.src || '';

            let imageUrl = srcset ?
                srcset.split(',')
                      .pop()
                      .trim()
                      .split(' ')[0] :
                src;

            if (imageUrl) {
                count3++;
                let imageUrl2 = "";
                if (window.location.href.startsWith('https://www.wwhitestudio.com')) {
                    imageUrl2 = imageUrl.replace(/(\d+x\.JPG.*)/, 'original.JPG');
                }else {
                    imageUrl2 = imageUrl.replace(/(\d+x)\.[a-z].*/, 'original.jpg');
                }
                extractedImages.push(imageUrl2);
                let imageThumbs = "";
                if (window.location.href.startsWith('https://www.wwhitestudio.com')) {
                    imageThumbs = imageUrl2.replace(/original\.JPG$/, '140x.JPG');
                }else {
                    imageThumbs = imageUrl2.replace(/original\.jpg$/, '140x.webp?source_format=jpg');
                }
                extractedThumbs.push(imageThumbs);
            }
        });

        // Console log results
        console.log('Extraction results:');
        console.log(`1 (ProductGallery-item-image): ${count1} images extracted` + (count1 === 0 ? ' ✗' : ' ✓'));
        console.log(`2 (product-media--image): ${count2} images extracted` + (count2 === 0 ? ' ✗' : ' ✓'));
        console.log(`3 (ProductDetail-gallery): ${count3} images extracted` + (count3 === 0 ? ' ✗' : ' ✓'));
        console.log('Total extracted images:', extractedImages);
        console.log('Total extracted thumbnails:', extractedThumbs);

        if (extractedImages.length > 0) {
            let message = `All images extracted! (${extractedImages.length} total)`;
            showMessage(message);

            const showBtn = document.querySelector('#image-extractor-box button:last-child');
            showBtn.disabled = false;
            showBtn.style.background = '#28a745';
            showBtn.style.cursor = 'pointer';
            showBtn.onmouseover = () => showBtn.style.background = '#1e7e34';
            showBtn.onmouseout = () => showBtn.style.background = '#28a745';
        } else {
            showMessage('No images found! (All 3 extractors tried)');
        }
    }

    function showMessage(text) {
        const message = document.createElement('div');
        message.textContent = text;
        message.style.cssText = `
            position: fixed;
            bottom: 110px;
            right: 20px;
            background: #333;
            color: white;
            padding: 10px 15px;
            border-radius: 5px;
            font-family: Arial, sans-serif;
            font-size: 14px;
            z-index: 10001;
            animation: fadeInOut 3s ease-in-out;
        `;

        const style = document.createElement('style');
        style.textContent = `
            @keyframes fadeInOut {
                0% { opacity: 0; transform: translateY(10px); }
                20% { opacity: 1; transform: translateY(0); }
                80% { opacity: 1; transform: translateY(0); }
                100% { opacity: 0; transform: translateY(-10px); }
            }
        `;
        document.head.appendChild(style);
        document.body.appendChild(message);

        setTimeout(() => {
            if (message.parentNode) {
                message.parentNode.removeChild(message);
            }
        }, 3000);
    }

    // Show images using OpenSeadragon
    function showImages() {
        if (extractedImages.length === 0) return;

        currentImageIndex = 0;
        createOverlay();
        initializeViewer();
    }

    // Open current image in new tab
    function openImageInNewTab() {
        const currentImageUrl = extractedImages[currentImageIndex];
        if (currentImageUrl) {
            window.open(currentImageUrl, '_blank');
        }
    }

    // Create overlay layer
    function createOverlay() {
        overlayContainer = document.createElement('div');
        overlayContainer.id = 'image-viewer-overlay';
        overlayContainer.style.cssText = `
            position: fixed;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            background: rgba(0, 0, 0, 0.9);
            z-index: 10002;
            display: flex;
            flex-direction: column;
        `;

        // Create header with controls
        const header = document.createElement('div');
        header.style.cssText = `
            background: rgba(0, 0, 0, 0.8);
            padding: 15px;
            display: flex;
            justify-content: space-between;
            align-items: center;
            color: white;
            font-family: Arial, sans-serif;
        `;

        const title = document.createElement('div');
        title.textContent = `Image 1 of ${extractedImages.length}`;
        title.id = 'image-counter';
        title.style.fontSize = '18px';

        const controls = document.createElement('div');
        controls.style.display = 'flex';
        controls.style.gap = '10px';

        // Previous button
        const prevBtn = document.createElement('button');
        prevBtn.textContent = '← Previous';
        prevBtn.style.cssText = `
            padding: 8px 16px;
            background: #007bff;
            color: white;
            border: none;
            border-radius: 5px;
            cursor: pointer;
        `;
        prevBtn.addEventListener('click', () => navigateImage(-1));

        // Next button
        const nextBtn = document.createElement('button');
        nextBtn.textContent = 'Next →';
        nextBtn.style.cssText = `
            padding: 8px 16px;
            background: #007bff;
            color: white;
            border: none;
            border-radius: 5px;
            cursor: pointer;
        `;
        nextBtn.addEventListener('click', () => navigateImage(1));

        // New Tab button
        const newTabBtn = document.createElement('button');
        newTabBtn.textContent = '↗ New Tab';
        newTabBtn.style.cssText = `
            padding: 8px 16px;
            background: #28a745;
            color: white;
            border: none;
            border-radius: 5px;
            cursor: pointer;
            transition: background 0.3s;
        `;
        newTabBtn.onmouseover = () => newTabBtn.style.background = '#1e7e34';
        newTabBtn.onmouseout = () => newTabBtn.style.background = '#28a745';
        newTabBtn.addEventListener('click', openImageInNewTab);

        // Save Current button
        const saveCurrent = document.createElement('button');
        saveCurrent.textContent = '💾 Save';
        saveCurrent.style.cssText = `
            padding: 8px 16px;
            background: #ffc107;
            color: white;
            border: none;
            border-radius: 5px;
            cursor: pointer;
        `;
        saveCurrent.addEventListener('click', () => {
            const currentImageUrl = extractedImages[currentImageIndex];
            if (currentImageUrl) {
                const link = document.createElement('a');
                link.href = currentImageUrl;
                const paddedIndex = String(currentImageIndex + 1).padStart(5, '0');
                const timestamp = new Date().toISOString().replace(/[:.]/g, '-');
                link.download = `image-${paddedIndex}-${timestamp}.jpg`;
                link.target = '_blank';
                document.body.appendChild(link);
                link.click();
                document.body.removeChild(link);
                showMessage(`Image ${currentImageIndex + 1} saved!`);
            }
        });

        // Fullscreen button
        const fullscreenBtn = document.createElement('button');
        fullscreenBtn.textContent = '⛶ Fullscreen';
        fullscreenBtn.style.cssText = `
            padding: 8px 16px;
            background: #6f42c1;
            color: white;
            border: none;
            border-radius: 5px;
            cursor: pointer;
        `;
        fullscreenBtn.addEventListener('click', toggleFullscreen);

        // Close button
        const closeBtn = document.createElement('button');
        closeBtn.textContent = '✕ Close';
        closeBtn.style.cssText = `
            padding: 8px 16px;
            background: #dc3545;
            color: white;
            border: none;
            border-radius: 5px;
            cursor: pointer;
        `;
        closeBtn.addEventListener('click', closeOverlay);

        controls.appendChild(prevBtn);
        controls.appendChild(nextBtn);
        controls.appendChild(newTabBtn);
        controls.appendChild(saveCurrent);
        controls.appendChild(fullscreenBtn);
        controls.appendChild(closeBtn);

        header.appendChild(title);
        header.appendChild(controls);

        // Create main content area with thumbnails and viewer
        const mainContent = document.createElement('div');
        mainContent.id = 'main-content';

        // Check screen width for responsive layout
        if (screen.width <= 1280) {
            mainContent.style.cssText = `
                flex: 1;
                display: flex;
                flex-direction: column-reverse;
                height: calc(100% - 70px);
            `;
        } else {
            mainContent.style.cssText = `
                flex: 1;
                display: flex;
                height: calc(100% - 70px);
            `;
        }

        // Create viewer container
        const viewerContainer = document.createElement('div');
        viewerContainer.id = 'openseadragon-viewer';
        viewerContainer.style.cssText = `
            flex: 1;
            width: 100%;
            height: ${screen.width <= 1280 ? 'calc(100% - 150px)' : '100%'};
        `;

        // Create thumbnail panel
        const thumbnailPanel = document.createElement('div');
        thumbnailPanel.id = 'thumbnail-panel';
        if (screen.width <= 1280) {
            thumbnailPanel.style.cssText = `
                height: 150px;
                background: rgba(0, 0, 0, 0.8);
                border-top: 1px solid #333;
                overflow-x: auto;
                overflow-y: hidden;
                padding: 10px;
                display: flex;
                gap: 8px;
                flex-direction: row;
            `;
        } else {
            thumbnailPanel.style.cssText = `
                width: 200px;
                background: rgba(0, 0, 0, 0.8);
                border-right: 1px solid #333;
                overflow-y: auto;
                padding: 10px;
                display: flex;
                flex-direction: column;
                gap: 8px;
            `;
        }

        // Create thumbnails
        extractedThumbs.forEach((imageThumbs, index) => {
            const thumbnailContainer = document.createElement('div');
            thumbnailContainer.style.cssText = `
                position: relative;
                cursor: pointer;
                border: 2px solid ${index === 0 ? '#007bff' : 'transparent'};
                border-radius: 4px;
                transition: border-color 0.3s;
                ${screen.width <= 1280 ? 'flex: 0 0 auto;' : ''}
            `;
            thumbnailContainer.dataset.index = index;

            const thumbnail = document.createElement('img');
            thumbnail.src = imageThumbs;
            if (screen.width <= 1280) {
                thumbnail.style.cssText = `
                    height: 120px;
                    width: auto;
                    object-fit: cover;
                    border-radius: 2px;
                    display: block;
                `;
            } else {
                thumbnail.style.cssText = `
                    width: 100%;
                    height: 120px;
                    object-fit: cover;
                    border-radius: 2px;
                    display: block;
                `;
            }

            const imageNumber = document.createElement('div');
            imageNumber.textContent = index + 1;
            imageNumber.style.cssText = `
                position: absolute;
                top: 4px;
                left: 4px;
                background: rgba(0, 0, 0, 0.7);
                color: white;
                padding: 2px 6px;
                border-radius: 3px;
                font-size: 12px;
                font-family: Arial, sans-serif;
            `;

            thumbnailContainer.appendChild(thumbnail);
            thumbnailContainer.appendChild(imageNumber);

            // Add click event to thumbnail
            thumbnailContainer.addEventListener('click', () => {
                navigateToImage(index);
            });

            // Add hover effect
            thumbnailContainer.addEventListener('mouseenter', () => {
                if (index !== currentImageIndex) {
                    thumbnailContainer.style.borderColor = '#6c757d';
                }
            });

            thumbnailContainer.addEventListener('mouseleave', () => {
                if (index !== currentImageIndex) {
                    thumbnailContainer.style.borderColor = 'transparent';
                }
            });

            thumbnailPanel.appendChild(thumbnailContainer);
        });

        if (screen.width <= 1280) {
            mainContent.appendChild(viewerContainer);
            mainContent.appendChild(thumbnailPanel);
        } else {
            mainContent.appendChild(thumbnailPanel);
            mainContent.appendChild(viewerContainer);
        }

        overlayContainer.appendChild(header);
        overlayContainer.appendChild(mainContent);
        document.body.appendChild(overlayContainer);

        // Add escape key listener
        document.addEventListener('keydown', handleKeyPress);
    }

    // Toggle fullscreen mode
    function toggleFullscreen() {
        if (!isFullScreen) {
            // Enter fullscreen
            overlayContainer.style.position = 'fixed';
            overlayContainer.style.top = '0';
            overlayContainer.style.left = '0';
            overlayContainer.style.width = '100vw';
            overlayContainer.style.height = '100vh';
            overlayContainer.style.zIndex = '10003';

            // Hide thumbnails in fullscreen mode
            document.getElementById('thumbnail-panel').style.display = 'none';

            // Maximize viewer
            const viewerContainer = document.getElementById('openseadragon-viewer');
            viewerContainer.style.width = '100%';
            viewerContainer.style.height = '100%';

            isFullScreen = true;
        } else {
            // Exit fullscreen
            overlayContainer.style.position = '';
            overlayContainer.style.top = '';
            overlayContainer.style.left = '';
            overlayContainer.style.width = '';
            overlayContainer.style.height = '';
            overlayContainer.style.zIndex = '';

            // Show thumbnails again
            const thumbnailPanel = document.getElementById('thumbnail-panel');
            thumbnailPanel.style.display = screen.width <= 1280 ? 'flex' : 'block';

            // Restore viewer size
            const viewerContainer = document.getElementById('openseadragon-viewer');
            viewerContainer.style.width = screen.width <= 1280 ? '100%' : 'calc(100% - 200px)';
            viewerContainer.style.height = screen.width <= 1280 ? 'calc(100% - 150px)' : '100%';

            isFullScreen = false;
        }

        // Force OpenSeadragon to resize
        if (viewer) {
            setTimeout(() => {
                viewer.viewport.resize();
                viewer.viewport.fitBounds(viewer.world.getItemAt(0).getBounds());
            }, 100);
        }
    }

    // Initialize OpenSeadragon viewer
    function initializeViewer() {
        viewer = OpenSeadragon({
            id: 'openseadragon-viewer',
            prefixUrl: 'https://cdnjs.cloudflare.com/ajax/libs/openseadragon/4.1.0/images/',
            tileSources: {
                type: 'image',
                url: extractedImages[currentImageIndex]
            },
            showNavigationControl: true,
            showZoomControl: true,
            showHomeControl: true,
            showFullPageControl: false, // We're using our own fullscreen button
            minZoomLevel: 0.5,
            maxZoomLevel: 10,
            zoomInButton: 'zoom-in',
            zoomOutButton: 'zoom-out',
            homeButton: 'home',
            fullPageButton: 'full-page'
        });
    }

    // Navigate between images
    function navigateImage(direction) {
        const newIndex = currentImageIndex + direction;
        if (newIndex >= 0 && newIndex < extractedImages.length) {
            navigateToImage(newIndex);
        }
    }

    // Navigate to specific image index
    function navigateToImage(index) {
        if (index >= 0 && index < extractedImages.length && index !== currentImageIndex) {
            // Update thumbnail selection
            updateThumbnailSelection(index);

            currentImageIndex = index;

            // Update counter
            document.getElementById('image-counter').textContent =
                `Image ${currentImageIndex + 1} of ${extractedImages.length}`;

            // Update viewer
            viewer.open({
                type: 'image',
                url: extractedImages[currentImageIndex]
            });

            // Scroll thumbnail into view
            scrollThumbnailIntoView(index);
        }
    }

    // Update thumbnail selection visual state
    function updateThumbnailSelection(newIndex) {
        const thumbnails = document.querySelectorAll('#thumbnail-panel > div');

        // Remove selection from current thumbnail
        if (thumbnails[currentImageIndex]) {
            thumbnails[currentImageIndex].style.borderColor = 'transparent';
        }

        // Add selection to new thumbnail
        if (thumbnails[newIndex]) {
            thumbnails[newIndex].style.borderColor = '#007bff';
        }
    }

    // Scroll thumbnail into view
    function scrollThumbnailIntoView(index) {
        const thumbnailPanel = document.getElementById('thumbnail-panel');
        const thumbnails = thumbnailPanel.querySelectorAll('div');

        if (thumbnails[index]) {
            const thumbnail = thumbnails[index];

            if (screen.width <= 1280) {
                // Horizontal scrolling for small screens
                const panelWidth = thumbnailPanel.clientWidth;
                const thumbnailLeft = thumbnail.offsetLeft;
                const thumbnailWidth = thumbnail.offsetWidth;
                const currentScroll = thumbnailPanel.scrollLeft;

                if (thumbnailLeft < currentScroll || thumbnailLeft + thumbnailWidth > currentScroll + panelWidth) {
                    // Scroll to center the thumbnail horizontally
                    thumbnailPanel.scrollLeft = thumbnailLeft - (panelWidth / 2) + (thumbnailWidth / 2);
                }
            } else {
                // Vertical scrolling for larger screens
                const panelHeight = thumbnailPanel.clientHeight;
                const thumbnailTop = thumbnail.offsetTop;
                const thumbnailHeight = thumbnail.offsetHeight;
                const currentScroll = thumbnailPanel.scrollTop;

                if (thumbnailTop < currentScroll || thumbnailTop + thumbnailHeight > currentScroll + panelHeight) {
                    // Scroll to center the thumbnail vertically
                    thumbnailPanel.scrollTop = thumbnailTop - (panelHeight / 2) + (thumbnailHeight / 2);
                }
            }
        }
    }

    // Handle keyboard navigation
    function handleKeyPress(event) {
        if (!overlayContainer) return;

        switch(event.key) {
            case 'Escape':
                if (isFullScreen) {
                    toggleFullscreen();
                } else {
                    closeOverlay();
                }
                break;
            case 'ArrowLeft':
                navigateImage(-1);
                break;
            case 'ArrowRight':
                navigateImage(1);
                break;
            case 'n':
            case 'N':
                // Press 'N' to open image in new tab
                openImageInNewTab();
                break;
            case 'f':
            case 'F':
                // Press 'F' to toggle fullscreen
                toggleFullscreen();
                break;
        }
    }

    // Close overlay
    function closeOverlay() {
        if (viewer) {
            viewer.destroy();
            viewer = null;
        }
        if (overlayContainer) {
            overlayContainer.remove();
            overlayContainer = null;
        }
        document.removeEventListener('keydown', handleKeyPress);
        isFullScreen = false;
    }

    // Initialize the script
    function init() {
        // Wait for page to load
        if (document.readyState === 'loading') {
            document.addEventListener('DOMContentLoaded', createFloatingBox);
        } else {
            createFloatingBox();
        }
    }

    init();
})();
