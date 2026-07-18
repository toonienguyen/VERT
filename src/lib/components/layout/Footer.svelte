<script lang="ts">
	const year = new Date().getFullYear();
</script>

<footer
	class="hidden md:block w-full h-14 border-t border-separator fixed bottom-0 mt-12"
>
	<div
		class="w-full h-full flex items-center justify-center text-muted gap-3 relative"
	>
		<!-- Chỉ giữ lại đúng dòng chữ bản quyền, tự động đổi chữ VERT thành 𝕍𝔼ℝ𝕋 𝕋𝕆𝕆𝕃𝕊 nếu anh thích -->
		<p>© {year} 𝕍𝔼ℝ𝕋 𝕋𝕆𝕆𝕃𝕊</p>
	</div>

	<div
		class="absolute bottom-0 left-0 w-full h-24 -z-10 pointer-events-none"
		style="background: linear-gradient(to bottom, transparent, var(--bg) 100%)"
	></div>
</footer>
