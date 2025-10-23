<template>
    <div class="gallery-shell" @keydown="handleKeydown" tabindex="0">
        <!-- Header -->
        <header class="gallery-header">
            <div class="title">
                <h1>🌿 Image Gallery</h1>
                <p class="subtitle">Masonry — responsive & interactive</p>
            </div>
            <div class="controls">
                <div class="action-row">
                    <input v-model="filterQuery" class="search" placeholder="Search title or tag..." />
                    <button v-if="deletedImages.length" class="restore" @click="restoreImage">🔄 Restore</button>
                </div>
            </div>
        </header>

        <!-- Masonry Grid -->
        <section class="masonry-grid">
            <div v-for="(img, idx) in filteredImages" :key="img.id" class="masonry-item">
                <div class="masonry-card" @click="selectImage(idx)">
                    <img :src="img.src" :alt="img.alt" @load="updateMasonry" loading="lazy" />
                    <div class="masonry-overlay">
                        <div class="masonry-info">
                            <h4>{{ img.alt }}</h4>
                            <p>{{ img.tags.join(', ') }}</p>
                        </div>
                        <button class="masonry-delete" @click.stop="deleteImage(idx)">🗑</button>
                    </div>
                </div>
            </div>
        </section>

        <p v-if="filteredImages.length === 0" class="empty">No images match your filters.</p>

        <!-- Lightbox -->
        <div v-if="isLightboxActive" class="lightbox" ref="lightbox" @touchstart="onTouchStart" @touchend="onTouchEnd">
            <div class="lb-backdrop" @click="closeLightbox"></div>
            <div class="lb-content">
                <button class="lb-close" @click="closeLightbox">✖</button>
                <button class="nav left" @click="prevImage">‹</button>

                <figure class="lb-figure">
                    <img :src="selectedImage.src" :alt="selectedImage.alt" class="lb-img"
                        :style="{ transform: 'scale(' + zoom + ')' }" @wheel.prevent="zoomImage" />
                    <figcaption class="lb-caption">
                        <h3>{{ selectedImage.alt }}</h3>
                        <p class="lb-tags">{{ selectedImage.tags.join(' • ') }}</p>
                        <div class="lb-pos">{{ currentIndex + 1 }} / {{ filteredImages.length }}</div>
                    </figcaption>
                </figure>

                <button class="nav right" @click="nextImage">›</button>
                <div class="zoom-controls">
                    <button @click="zoomIn">＋</button>
                    <button @click="zoomOut">－</button>
                    <button @click="toggleFit">⤢</button>
                    <button class="del lb-del" @click="deleteImage(currentIndex)">🗑</button>
                </div>
            </div>
        </div>
    </div>
</template>

<script>
export default {
    name: "ImageGallery",
    data() {
        const sampleCount = 9;
        const imgs = Array.from({ length: sampleCount }, (_, i) => ({
            id: i + 1,
            src: `/images/Image${i + 1}.png`,
            alt: `Image ${i + 1}`,
            tags: ["nature", i % 2 === 0 ? "landscape" : "scenery"],
        }));
        return {
            images: imgs,
            deletedImages: [],
            selectedImageIndex: null,
            zoom: 1,
            touchStartX: 0,
            filterQuery: "",
        };
    },
    computed: {
        filteredImages() {
            const q = this.filterQuery.trim().toLowerCase();
            if (!q) return this.images;
            return this.images.filter(
                (img) =>
                    img.alt.toLowerCase().includes(q) ||
                    img.tags.join(" ").toLowerCase().includes(q)
            );
        },
        isLightboxActive() {
            return this.selectedImageIndex !== null && this.filteredImages[this.selectedImageIndex];
        },
        selectedImage() {
            return this.filteredImages[this.selectedImageIndex] || {};
        },
        currentIndex() {
            return this.selectedImageIndex ?? 0;
        },
    },
    methods: {
        selectImage(index) {
            this.selectedImageIndex = index;
            this.zoom = 1;
            this.focusLightbox();
        },
        closeLightbox() {
            this.selectedImageIndex = null;
            this.zoom = 1;
            this.$el && this.$el.focus();
        },
        deleteImage(index) {
            const img = this.filteredImages[index];
            if (!img) return;
            const pos = this.images.findIndex((i) => i.id === img.id);
            if (pos === -1) return;
            if (this.isLightboxActive && this.currentIndex === index) this.closeLightbox();
            const removed = this.images.splice(pos, 1)[0];
            this.deletedImages.push(removed);
            this.$nextTick(this.updateMasonry);
        },
        restoreImage() {
            if (!this.deletedImages.length) return;
            const r = this.deletedImages.pop();
            this.images.push(r);
            this.$nextTick(this.updateMasonry);
        },
        prevImage() {
            if (!this.isLightboxActive) return;
            this.selectedImageIndex =
                this.selectedImageIndex > 0 ? this.selectedImageIndex - 1 : this.filteredImages.length - 1;
            this.zoom = 1;
        },
        nextImage() {
            if (!this.isLightboxActive) return;
            this.selectedImageIndex =
                this.selectedImageIndex < this.filteredImages.length - 1 ? this.selectedImageIndex + 1 : 0;
            this.zoom = 1;
        },
        zoomIn() { this.zoom = Math.min(this.zoom + 0.15, 3); },
        zoomOut() { this.zoom = Math.max(this.zoom - 0.15, 0.5); },
        zoomImage(e) {
            this.zoom += e.deltaY > 0 ? -0.1 : 0.1;
            this.zoom = Math.max(0.5, Math.min(3, this.zoom));
        },
        toggleFit() { this.zoom = 1; },
        handleKeydown(e) {
            if (!this.isLightboxActive) return;
            switch (e.key) {
                case "ArrowLeft": this.prevImage(); break;
                case "ArrowRight": this.nextImage(); break;
                case "Escape": this.closeLightbox(); break;
                case "Delete": this.deleteImage(this.selectedImageIndex); break;
                case "ArrowUp": this.zoomIn(); break;
                case "ArrowDown": this.zoomOut(); break;
            }
        },
        focusLightbox() { this.$nextTick(() => { const el = this.$refs.lightbox; el && el.focus(); }); },
        onTouchStart(e) { this.touchStartX = e.changedTouches[0].screenX; },
        onTouchEnd(e) {
            const diff = e.changedTouches[0].screenX - this.touchStartX;
            if (Math.abs(diff) > 40) diff > 0 ? this.prevImage() : this.nextImage();
        },
        updateMasonry() {
            this.$nextTick(() => {
                const grid = this.$el.querySelector('.masonry-grid');
                if (!grid) return;
                const rowHeight = parseInt(getComputedStyle(grid).getPropertyValue('grid-auto-rows'));
                const rowGap = parseInt(getComputedStyle(grid).getPropertyValue('gap'));
                const items = grid.querySelectorAll('.masonry-item');

                items.forEach(item => {
                    const img = item.querySelector('img');
                    if (img.complete) {
                        const span = Math.ceil((img.getBoundingClientRect().height + rowGap) / (rowHeight + rowGap));
                        item.style.gridRowEnd = `span ${span}`;
                    }
                });
            });
        }
    },
    mounted() {
        this.updateMasonry();
        window.addEventListener('resize', this.updateMasonry);
        this.$el && this.$el.focus();
    },
    beforeUnmount() {
        window.removeEventListener('resize', this.updateMasonry);
    }
};
</script>

<style scoped>
/* ===== Shell & Header ===== */
.gallery-shell {
    max-width: 1200px;
    margin: 24px auto;
    padding: 18px;
    font-family: 'Inter', system-ui;
    color: #0f1724;
}

.gallery-header {
    display: flex;
    flex-wrap: wrap;
    gap: 12px;
    justify-content: space-between;
    margin-bottom: 14px;
    align-items: center;
}

.title h1 {
    margin: 0;
    font-size: 1.4rem;
}

.subtitle {
    margin: 2px 0 0;
    font-size: 0.85rem;
    color: #6b7280;
}

.controls {
    display: flex;
    flex-wrap: wrap;
    gap: 12px;
    align-items: center;
}

.action-row {
    display: flex;
    gap: 8px;
    align-items: center;
}

.search {
    padding: 8px 12px;
    border-radius: 10px;
    border: 1px solid #e6e7ea;
    min-width: 200px;
}

.restore {
    background: #10b981;
    color: #fff;
    border: none;
    padding: 8px 12px;
    border-radius: 10px;
    cursor: pointer;
}

/* ===== Masonry Grid ===== */
.masonry-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(220px, 1fr));
    grid-auto-rows: 10px;
    /* base row height */
    gap: 16px;
}

.masonry-item {
    border-radius: 16px;
    overflow: hidden;
    position: relative;
}

.masonry-card {
    position: relative;
    cursor: pointer;
    overflow: hidden;
    border-radius: 16px;
    background: #fff;
    box-shadow: 0 8px 30px rgba(15, 23, 42, 0.08);
    transition: 0.3s;
}

.masonry-card img {
    width: 100%;
    display: block;
    border-radius: 16px;
    object-fit: cover;
    transition: transform 0.3s;
}

.masonry-overlay {
    position: absolute;
    inset: 0;
    display: flex;
    flex-direction: column;
    justify-content: flex-end;
    opacity: 0;
    background: linear-gradient(180deg, rgba(0, 0, 0, 0)50%, rgba(0, 0, 0, 0.65)100%);
    transition: opacity 0.3s ease;
    padding: 12px;
}

.masonry-card:hover .masonry-overlay {
    opacity: 1;
}

.masonry-info h4 {
    margin: 0;
    color: #fff;
    font-size: 1rem;
    font-weight: 600;
}

.masonry-info p {
    margin: 4px 0 0;
    font-size: 0.78rem;
    color: rgba(255, 255, 255, 0.85);
}

.masonry-delete {
    position: absolute;
    top: 8px;
    right: 8px;
    background: rgba(255, 0, 0, 0.2);
    color: #fff;
    border: none;
    border-radius: 8px;
    padding: 4px 6px;
    cursor: pointer;
}

.masonry-delete:hover {
    background: rgba(255, 0, 0, 0.4);
}

/* ===== Lightbox ===== */
.lightbox {
    position: fixed;
    inset: 0;
    background: rgba(0, 0, 0, 0.85);
    display: flex;
    justify-content: center;
    align-items: center;
    z-index: 999;
    outline: none;
    gap: 20px;
}

.lb-backdrop {
    position: absolute;
    inset: 0;
}

.lb-content {
    position: relative;
    display: flex;
    justify-content: center;
    align-items: center;
    gap: 12px;
}

.lb-figure {
    margin: 0;
    display: flex;
    flex-direction: column;
    align-items: center;
}

.lb-img {
    max-height: 80vh;
    max-width: 80vw;
    border-radius: 12px;
    transition: transform 0.3s ease;
}

.lb-caption {
    text-align: center;
    color: #fff;
    margin-top: 8px;
    font-size: 0.85rem;
}

.lb-tags {
    font-size: 0.75rem;
    opacity: 0.75;
}

.lb-pos {
    font-size: 0.7rem;
    margin-top: 2px;
}

.lb-close {
    position: absolute;
    top: 8px;
    right: 8px;
    background: #ef4444;
    border: none;
    color: #fff;
    font-size: 1rem;
    border-radius: 6px;
    padding: 4px 6px;
    cursor: pointer;
}

.nav {
    background: none;
    color: #fff;
    font-size: 3rem;
    border: none;
    cursor: pointer;
    user-select: none;
    transition: color 0.25s;
}

.nav:hover {
    color: #10b981;
}

.nav.left {
    position: absolute;
    left: 8px;
    top: 50%;
    transform: translateY(-50%);
}

.nav.right {
    position: absolute;
    right: 8px;
    top: 50%;
    transform: translateY(-50%);
}

.zoom-controls {
    position: absolute;
    bottom: 16px;
    display: flex;
    gap: 6px;
    justify-content: center;
    width: 100%;
}

.zoom-controls button {
    background: rgba(0, 0, 0, 0.4);
    color: #fff;
    border: none;
    padding: 6px 8px;
    border-radius: 6px;
    cursor: pointer;
    font-size: 1rem;
}

.zoom-controls button:hover {
    background: rgba(0, 0, 0, 0.6);
}

.lb-del {
    background: rgba(255, 0, 0, 0.6);
}

.empty {
    text-align: center;
    margin-top: 40px;
    color: #6b7280;
    font-size: 0.95rem;
}

/* ===== Responsive ===== */
@media (max-width:1100px) {
    .masonry-grid {
        grid-template-columns: repeat(auto-fill, minmax(180px, 1fr));
    }
}

@media (max-width:800px) {
    .masonry-grid {
        grid-template-columns: repeat(auto-fill, minmax(150px, 1fr));
    }
}

@media (max-width:520px) {
    .masonry-grid {
        grid-template-columns: repeat(auto-fill, minmax(120px, 1fr));
    }
}
</style>
