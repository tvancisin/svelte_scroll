<script>
	export let center = true;
	let width;
</script>

<nav style="background-color: white;" bind:clientWidth={width}>
  <!-- <div class="col-wide middle" class:center> -->
  <div style="width:{width}px" class:center>
		<a href="https://peacerep.org/">
			<picture>
				<img src="./img/PeaceRep_nobg.png" alt="Office for National Statistics">
			</picture>
		</a>
  </div>
</nav>

<style>
	nav {
		-webkit-box-sizing: border-box;
		box-sizing: border-box;
		position: relative;
		height: 50px;
		margin-bottom: -49px;
		z-index: 1;
	}
	picture {
		position: relative;
		top: 1px;
		left: 1px;
	}
	img {
		height: 49px;
	}
	a img:hover {
		cursor: pointer;
	}
</style>