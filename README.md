# tudat-matlab
MATLAB interface for Tudat

### Installation

1. Clone or download [tudatBundle-json](http://github.com/aleixpinardell/tudatBundle/tree/json) (see [how to to this](http://tudat.tudelft.nl/installation/)).
2. Compile the target `tudat` with QT Creator.
3. Clone or download this repository.
4. Run the MATLAB script `quickinstall.m`.
5. You will be prompted to provide the path to the `tudat` binary generated in step 2. You can skip this step by typing `ctrl + C` if you are planning to use tudat-matlab only to generate input files (and then run `tudat` from the command line, from QT Creator or on the server). You can specify `tudat` binary path later by calling `tudat.locate('binaryPath')` from MATLAB's Command Window.

### Usage

In this section, the steps to simulate the unperturbed motion of a satellite about the Earth will be described.

The first step is to include the paths to tudat-matlab source code in the current MATLAB session so that you can use all the classes needed to set up the simulation. You do this by writing in a new script:
```
tudat.load();
```

Now, you can create a `Simulation` object by writing:
```
simulation = Simulation('1992-02-14 07:30','1992-02-15 07:30');
```

If you want to load automatically the ephemeris and properties of bodies such as the Sun, Earth, the Moon and other planets, you will need to use Spice. For a simple propagation, you do this by speciying the following Spice kernels:
```
simulation.spice = Spice('pck00009.tpc','de-403-masses.tpc','de421.bsp');
```

Next, you need to create the bodies. For an unperturbed orbit, the mass of the satellite is irrelevant, so creating an empty body object (named 'Satellite') will suffice;
```
satelliteBody = Body('Satellite');
```

Now, you add the bodies to the propagation by calling the method `addBodies` of your `simulation` object. You have to provide a list of `Body` objects and or body names. When you provide a body name (i.e. a string), a `Body` object will be created for you and its properties will be retrieved from Spice (if its name is recognizable). Thus, you can simply write:
```
simulation.addBodies('Earth',satelliteBody);
```

Then, you will create the settings for the propagation. We are going to propagate the translational state of the body 'Satellite' about the 'Earth'. Thus, we will use a `TranslationalPropagator`:
```
propagator = TranslationalPropagator();
initialKeplerianState = [7500.0E3 0.1 deg2rad(85.3) deg2rad(235.7) deg2rad(23.4) deg2rad(139.87)];
propagator.initialState = convert.keplerianToCartesian(initialKeplerianState);
propagator.centralBody = 'Earth';
propagator.bodyToPropagate = 'Satellite';
```

Note that we always refer to bodies by their names (i.e. we do not provide the `Body` object `satelliteBody` to `propagator.bodyToPropagate`). The only exception is when calling the method `addBodies` of a `Simulation` object.

Also note the usage of tudat-matlab's `convert` package, which includes a few useful function for conversion of units and orbital elements.

Now we need to specify the accelerations acting on 'Satellite'. The only accelerations acting on satellite are those caused by 'Earth', so we need to specify the property `propagator.accelerations.Asterix.Earth`. We assign a list (i.e. a cell array) of `Acceleration` objects. In the case of an unperturbed satellite, the only acceleration it the point-mass gravity of the central body. Since `PointMassGravity` is a derived class of `Acceleration`, we can write:
```
propagator.accelerations.Asterix.Earth = { PointMassGravity() };
```

Finally, we add the `propagator` to the `simulation` object and define an integrator:
```
simulation.propagation = propagator;
simulation.integrator = Integrator(Integrators.rungeKutta4,20);
```
In this case we use a Runge-Kutta 4 integrator with a fixed step-size of 20 seconds.

Now, the simulation is set up and you can proceed in two different ways.
* **SL mode** (seamless mode). You use tudat-matlab to set up simulations that will be run directly from your MATLAB script. Temporary input and output files will be genereated and deleted by tudat-matlab in the background. When the simulation completes, you will be able to access the results directly in the property `results` of your `Simulation` object. You can use this data to generate plots and eventually consolidate (parts of) it in a text file.
* **IO mode** (input-output mode). You use tudat-matlab to set up simulations and to generate JSON input files that will then be provided to the `tudat` binary. Then, the generated output files can be opened with your favorite text editor and/or loaded into MATLAB for post-processing and plotting.

These modes are not mutually exclusive, i.e. you can run your simulations from MATLAB and still get to keep the input and output files in your directory. In the following two sub-sections, the two modes are briefly described.


#### SL mode

After setting up your simulation by following the steps described in [Usage](#usage), the only thing you have to do is write:
```
simulation.run();
```

Now, you are able to access the requested results from the `results` property of your simulation object. In addition to the requested results (in this case no results were requested), you are always able to access the property `results.numericalSolution`, which is a matrix in which each row corresponds to an integration step. The first column contains the value of the independent variable (the epoch in this case) and the other columns contain the state (the Cartesian component of 'Satellite'). You can decompose this matrix into epoch, position and velocity by writing:
```
[t,r,v] = compute.epochPositionVelocity(simulation.results.numericalSolution);
```

Finally, you can run MATLAB command on your results as usual:
```
plot(convert.epochToDate(t),r);
```


#### IO mode

After setting up your simulation by following the steps described in [Usage](#usage), you have to specify the output files that you want to generate. Otherwise, when running `tudat`, the simulation will be completed but no output will be generated. You do this by writing:
```
simulation.addResultsToExport('results.txt',{'independent','state'});
```

In this case, after the simulation is completed, a text file will be generated, containing a matrix in which each row will correspond to an integration step. The first column will contain the value of the independent variable (the epoch in this case) and the other columns will contain the state (the Cartesian component of 'Satellite').

Optionally, you can specify to generate an input file containing all the data loaded from Spice and all the default values used for keys that have not been specified, also known as a populated input file (note that this file will be generated when you run `tudat`). You do this by writing:

```
simulation.options.populatedFile = 'unperturbedSatellite-populated.json';
```

Now, you need to generate the JSON input file that will be provided to the `tudat` binary as command-line argument. You do this by using the `json` package of tudat-matlab:
```
json.export(simulation,'unperturbedSatellite.json');
```

At this point, after running your script, you can run `tudat` with the file 'unperturbedSatellite.json' generated in your working directory, by writing in the command line:
```
tudatBinaryPath unperturbedSatelliteInputFilePath
```

The files 'unperturbedSatellite-populated.json' and 'results.txt' will be generated next to your 'unperturbedSatellite.json'. Now, in MATLAB, you can post-process the results as usual:
```
results = load('results.txt');
t = results(:,1);
r = results(:,2:4);
plot(convert.epochToDate(t),r);
```
Note the usage of the function `epochToDate` from tudat-matlab's `convert` package, which converts seconds from J2000 to a MATLAB `datetime`.


