
CUDA Pathtracer (Fall 2013 CIS565 Project 2)
============================================

(Due Wednesday, 10/02/13)  
(Note: code and rendering in progress)

Full global illumination (note soft shadows and color bleeding)
![Diffuse](https://raw.github.com/takfuruya/Project2-Pathtracer/master/renders/color_bleeding.0.8.bmp)

Perfect specular reflection
![Reflective](https://raw.github.com/takfuruya/Project2-Pathtracer/master/renders/color_bleeding.0.10.bmp)

Supersampled antialiasing
![Anti-aliasing](https://raw.github.com/takfuruya/Project2-Pathtracer/master/anti-aliasing.png)

-------------------------------------------------------------------------------
Performance Analysis
-------------------------------------------------------------------------------
Program was parallelized by ray (instead of by pixel) via stream compaction.  

***In progress.***

-------------------------------------------------------------------------------
Nsight Insights
-------------------------------------------------------------------------------

My program was running fine when "Generate GPU Debug Information" was set to "Yes" but immediately
stopped execution when it was set to "No". No changes in code. What is happening?

![Generate GPU Debug Information](https://raw.github.com/takfuruya/Project2-Pathtracer/master/1.png)

I identified the issue on the host by running the following code after each kernel launch:

	cudaError_t err = cudaGetLastError();
	if( cudaSuccess != err) {
		fprintf(stderr, "Cuda error: %s.\n", cudaGetErrorString(err)); 
		exit(EXIT_FAILURE);
	}

I found that my path tracer kernel was not launching due to "too many resources requested for launch".
First, I checked the number of threads launched:

	int num_threads_per_block = 512;
	int num_blocks_per_grid = ceil((float)num_rays / num_threads_per_block);
	TraceRay<<<num_blocks_per_grid, num_threads_per_block>>>(...

I was working on a device with compute capability 1.2 (GeForce 310M) whose specs are:

 - max 512 threads/block
 - max 65535 blocks/grid
 - 2 SM
 - 8 blocks/SM

I am using ```ceil(800*800 / 512) = 1250``` blocks per grid so I am within limits.
Then I checked the amount of memory used in this kernel by adding "ptxas" flag to nvcc.

![ptxas Flag](https://raw.github.com/takfuruya/Project2-Pathtracer/master/2.png)

Building it, I was able to confirm that ```TraceRay``` kernel was using 63 registers.

![63 Registers](https://raw.github.com/takfuruya/Project2-Pathtracer/master/3.png)

	(63 registers/thread) * (512 threads/block) = 32256 registers/block

My device has limitation of 16384 registers per block so I guessed this was the issue:  
(Nsight > Windows > System Info)

![Max Registers Per Block](https://raw.github.com/takfuruya/Project2-Pathtracer/master/4.png)

I changed ```num_threads_per_block``` to 128 such that,

	(63 registers/thread) * (128 threads/block) = 8064 registers/block

...and it worked.  
It seems like setting "Generate GPU Debug Information" to "Yes" affects the amount of registers
that can be used per block.


-------------------------------------------------------------------------------
Extra
-------------------------------------------------------------------------------

A trippy image was produced when random number generator was not seeded properly.
![Max Registers Per Block](https://raw.github.com/takfuruya/Project2-Pathtracer/master/trippy.png)
