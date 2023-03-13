---
title: AM4 on Gaea
author: Tristan Abbott
categories:
- models
---

# Running AM4 on Gaea

## Compiling

Model compilation is configured as an "experiment" in XML files---but to be clear, this "experiment" does nothing but compile an executable. The XML required to configure compilation is relatively short. The file starts with a version tag and a notification about the GFDL fair use policy, and the rest of the file contents are wrapped in an `experimentSuite` tag:

```xml
<?xml version="1.0"?>

<!--
GFDL FAIR USE POLICY
... text omitted ...
-->

<experimentSuite rtsVersion="4" xmlns:xi="http://www.w3.org/2003/XInclude">
```

The first entry in the file determines which group's quota will be charged for compiling and running the model---so make sure you set the property to your group! I'm part of the M group, so I set it to `gfdl_m`:

```xml
  <!-- GFDL group. Should be changed to *your* group: gfdl_m, gfdl_o, etc. -->
  <property name="GFDL_YGL" value="gfdl_m"/>
```

The next entries control the source code versions that are used for compilation. The `RELEASE` property is used for FMS infrastructure, coupler, and atmosphere, ice, and land models. We're using the `xanadu` release, which is the version used for GFDL's CMIP6 runs. Depending on the component, `RELEASE` is either used to identify a git branch or a git tag. MOM6 (the ocean model) doesn't have a `xanadu` release, so `MOM6_RELEASE` is set separately to control the MOM6 code used.

```xml
  <!-- identifiers for source code versions -->
  <property name="RELEASE" value="xanadu"/>
  <property name="MOM6_RELEASE" value="dev/gfdl/2018.04.06"/>
```

The next entry controls the FMS Runtime Environment (FRE) version,

```xml
  <!-- identifier for FRE version -->
  <property name="FRE_VERSION" value="bronx-19"/>
```

and the final property entries,

```xml
  <!-- paths to XML and source files -->
  <property name="FRE_STEM" value="am4/xanadu"/>
  <property name="XMLDIR" value="xml"/>
  <property name="FMS_COMP_EXP" value="default"/>
```

are used elsewhere in the XML file to control directory structures. `FRE_STEM` is a "relative root" that distinguishes experiments with this model (AM4) and version (xanadu) from experiments with other models and model versions. `XMLDIR` is the directory where this XML file is located, and `FMS_COMP_EXP` is an identifier for the compilation "experiment".

Next, the XML file contains an entry with platform-specific configuration:

```xml
  <setup>
    <platform name="ncrc4.intel19">
      <freVersion>$(FRE_VERSION)</freVersion>
      <compiler type="intel" version="19.0.5.281" />
      <project>$(GFDL_YGL)</project>
      <directory stem="$(FRE_STEM)">
         <src>$HOME/$(stem)/$(name)/src</src>
      </directory>
      <property name="F2003_FLAGS"         value=" -DINTERNAL_FILE_NML "/>
      <csh><![CDATA[
        module load git
        module load gcp
        setenv KMP_STACKSIZE 512m
        setenv NC_BLKSZ 1M
        setenv OMP_SCHEDULE "static,1"
      ]]></csh>
    </platform>
  </setup>
```

The `name` attribute in the `platform` tag is matched against the `-p` flag provided when compiling with `fremake` and running with `frerun`. The `freVersion`, `compiler`, and `project` tags are fairly self-explanatory: they identify the FRE version used when compiling and running, the compiler used when compiling, and the project charged for the core hours used when compiling and running. The `directory` property determines paths used for I/O at runtime. For now, we require only a single entry that specifies where model source code will be stored. (Note that `$stem` is set in the `directory` tag, and `$name` is the experiment name provided through `fremake` or `frerun`. Other variables provided through `fremake` and `frerun` are `platform` (set by the `-p` flag) and `target` (set by the `-t` flag).) The `F2003_FLAGS` property provides a flag that is passed to the C preprocessor through an entry later in the file. Finally, the `csh` tag provides CSH code used to override platform defaults. These are included verbatim in an `env.cshrc` file created by `fremake` (on Gaea, at `$DEV/$USER/$FRE_STEM/$FMS_COMP_EXP/$(platform)-$(target)/exec`) that is sourced by the compilation script (created, on Gaea, in the same directory).

Next, the XML file contains an `experiment` tag that encloses information about the compilation "experiment":
```xml
  <experiment name="$(FMS_COMP_EXP)">

    <description>
        Experiment to build executable.
    </description>

    <component name="fms" paths="...">
        ...
    </component>
    
    <component name="atmos_phys" requires="fms"
     paths="atmos_param atmos_shared">
     <description domainName="" communityName=""
       communityVersion="$(RELEASE)" communityGrid=""/>
       <source versionControl="git" root="http://gitlab.gfdl.noaa.gov/fms">
         <codeBase version="master">atmos_shared.git atmos_param.git</codeBase>
         <csh><![CDATA[
           ( cd atmos_shared && git checkout  $(RELEASE) )
           ( cd atmos_param  && git checkout  $(RELEASE) )
         ]]>
         </csh>
       </source>
     <compile>
       <cppDefs>$(F2003_FLAGS) -DCLUBB </cppDefs>
     </compile>
    </component>

    ...

  </experiment>
```

The experiment name is set by the `FMS_COMP_EXP` property, and the `description` tag encloses a short description of the experiment. Each `component` contains instructions for checking out model code, and can also provide flags used at compile time. The full XML contains contains `component`s for FMS infrastructure, atmospheric physics, atmospheric dynamics, ice, land, ocean, and the coupler. I've included a stub for the FMS infrastructure and the full `component` entry for atmospheric physics, but omitted the others. (They are quite similar to the atmospheric physics `component`, and are available in the XML file linked at the end of the section.)

The atmospheric physics `component` tag contains a name for the component (`atmos_phys`), a list of other components it depends on (`requires="fms"`, referencing the FMS component defined above it), and a `paths` attribute that defines where code for the component will be stored. (These are used to generate build targets for each component, and the *user* is responsible for ensuring they match the names of folders, located inside the directory defined by the `src` tag, produced during the checkout process for the component.) The `description` tag within each `component` provides metadata about the component. The `source` tag includes attributes that control the VCS used for checkout and the root path used to identify repositories. The `codebase` tag contains a `version` attribute used to identify a git branch and encloses data used to identify git repositories to clone. The `csh` tag contains CSH code injected directly into the checkout script (`checkout.csh` in the `src` directory) and run as the final step during checkout. For the atmospheric physics component, the ultimate effect is to check out `xanadu` branches, albeit by first cloning master branches and then checking out `xanadu` branches. Finally, the `compile` tag is used to provide a list of flags passed to the C preprocessor during compilation. Other tags can be used within the `compile` tag to provide other effects at compile time, e.g., `<makeOverrides>VAR="VALUE"</makeOverrides>` to create or overwrite values in generated Makefiles.

One last line is required: a closing
```xml
</experimentSuite>
```
tag to match the tag at the start of the file.

A complete copy of this XML file, including all `component` tags, is available here.

To compile the model executable, switch to CSH, load the FRE module (`fre/bronx-19`, corresponding to `$FRE_VERSION`), and invoke `fremake`:

```console
$ csh
$ module load fre/bronx-19
$ fremake -x default.xml -p ncrc4.intel19 -t openmp default
```
The `-x` flag identifies this XML file, the `-p` flag identifies a `platform` tag for platform-specific configuration, and `-t openmp` tells `fremake` to compile with OpenMP support. (Not explicitly specifying `prod` (production, best performance), `debug` (debugging flags set), or `repro` (bitwise-reproducible results) implicitly selects `prod` as a target, so `$(platform)-$(target)` will be `ncrc4-intel19-prod-openmp`.) The final argument selects an experiment by name. This will check out model code to the path defined in `src` and create a compilation script at `$DEV/$USER/$FRE_STEM/$(name)/$(platform)-$(target)/exec/compile_$(name).csh`. Compilation can take a while, so to submit it as a batch job, run

```console
$ sbatch $DEV/$USER/am4/xanadu/default/ncrc4.intel19-prod-openmp/exec/compile_default.csh
```

If all goes well, a model executable named `fms_default.x` will be created in the same folder as the compile script.

## Running

We'll start by adding a new experiment (this time a real experiment that actually describes a model run) and then adding additional required information to the XML file.

Like the compilation experiment, this experiment begins with a name and a description. Additional, the `experiment` tag contains an `inherit` attribute that references the compilation experiment (maybe so the generated run script knows where to find the model executable)?

```xml
  <experiment name="c96L33_am4p0" inherit="$(FMS_COMP_EXP)">

    <description>
      AMIP experiment with UW two-plume convection
    </description>
```

Next, the experiment contains a `input` block that begins by specifying the locations of input namelists. The `namelist` blocks either specify a namelist directly (providing the name in the `name` attribute, and key-value pairs as data) or reference a file with a `file` attribute.

```xml
    <input>

      <namelist name="atmos_model_nml">
         nxblocks = 1
         nyblocks = $atm_threads
      </namelist>
      <namelist file="$(INPUT)/nml/ocean_g12.nml"/>
      <namelist file="$(INPUT)/nml/am4p0_common.nml"/>
      <namelist file="$(INPUT)/nml/am4p0_physics.nml"/>
      <namelist file="$(INPUT)/nml/am4p0_c96L33.nml"/>
      <namelist file="$(INPUT)/nml/am4p0_amip.nml"/>
      <namelist name="coupler_nml">
        months = $months,
        days   = $days,
        current_date = 1979,1,1,0,0,0,
        calendar = 'julian'
        dt_atmos = 1800,
        dt_cpld  = 7200,
        use_lag_fluxes = .true.
        concurrent = .false.
        do_ocean = .false.
        ocean_npes = $ocn_ranks
        atmos_npes = $atm_ranks
        atmos_nthreads = $atm_threads
        use_hyper_thread = $ht
        ncores_per_node = $(NCORE_PER_NODE)
      </namelist>
```

The `file` attribute uses two variables, `INPUT` and `NCORE_PER_NODE`, that we haven't defined yet. **TODO: figure out how other variables in namelist blocks are set.** The first variable provides the location of input files and the second provides information about the number of cores per node on the machine where the model will be run. Both may be platform-specific, so we'll define them inside the `platform` block. While we're there, we'll also define some additional standard paths in the `<directory>` block that control where model output and log files are stored:

```xml
  <setup>
    <platform name="ncrc4.intel19">
      <freVersion>$(FRE_VERSION)</freVersion>
      <compiler type="intel" version="19.0.5.281" />
      <project>$(GFDL_YGL)</project>
      <directory stem="$(FRE_STEM)">
        <src>$HOME/$(stem)/$(name)/src</src>
        <scripts>$HOME/$(stem)/scripts/$(platform)-$(target)</scripts>
        <stdout>$SCRATCH/$USER/$(stem)/$(FMS_COMP_EXP)/$(name)/$(platform)-$(target)/stdout</stdout>
        <state>$DEV/$USER/$(stem)/$(FMS_COMP_EXP)/$(name)/$(platform)-$(target)/state</state>
        <archive>$DEV/$USER/$(stem)/$(FMS_COMP_EXP)/$(name)/$(platform)-$(target)</archive>
        <postProcess>$DEV/$USER/$(stem)/$(FMS_COMP_EXP)/$(name)/$(platform)-$(target)/pp</postProcess>
        <analysis>$DEV/$USER/$(stem)/$(FMS_COMP_EXP)/$(name)/$(platform)-$(target)/analysis</analysis>
      </directory>
      <property name="F2003_FLAGS" value=" -DINTERNAL_FILE_NML "/>
      <property name="INPUT" value= "$(HOME)/$(FRE_STEM)/input"/>
      <property name="NCORE_PER_NODE" value="36"/>
      <csh><![CDATA[
        module load git
        module load gcp
        setenv KMP_STACKSIZE 512m
        setenv NC_BLKSZ 1M
        setenv OMP_SCHEDULE "static,1"
      ]]></csh>
    </platform>
  </setup>
```

The `scripts` path controls the location of run scripts produced by `frerun`. **TODO: figure out what other paths control.**

Finally, we need to create input namelists and place them in `$(INPUT)/nml`. For the sake of brevity, I'm assuming that you already have namelists configured with appropriate values and just need to figure out how to put them somewhere they'll be found by the model.

The `input` block continues by listing locations of input datasets:

```xml
      <xi:include xpointer="xpointer(//freInclude[@name='common_LM4']/input/node())"/>
      <xi:include xpointer="xpointer(//freInclude[@name='c96_LM4']/input/node())"/>
      <xi:include xpointer="xpointer(//freInclude[@name='common_AM4_v20170208']/input/node())"/>
      <xi:include xpointer="xpointer(//freInclude[@name='common_cmip6']/input/node())"/>
      <xi:include xpointer="xpointer(//freInclude[@name='Volcano_Solar_TimeSeries_cmip6']/input/node())"/>
      <xi:include xpointer="xpointer(//freInclude[@name='emissions_cmip6_v20161213']/input/node())"/>
      <xi:include xpointer="xpointer(//freInclude[@name='L32_gas_conc']/input/node())"/>
      <xi:include xpointer="xpointer(//freInclude[@name='amip_sst_cmip6']/input/node())"/>
```

Each dataset listed here must correspond to an `freInclude` block with matching `name`. Where you put them in the file doesn't matter as long as they're at the same level as the `experiment` tags. We'll add them at the bottom of the file. `common_LM4` provides information for the land model:

```xml
  <freInclude name="common_LM4">
    <input>
      <dataFile label="input" target="INPUT/soil_type.nc" chksum="" size="" timestamp="">
         <dataSource platform="$(platform)">$(CMIP6_ARCHIVE_ROOT)/CM4/common/common_LM4/soil_type_hwsd_5minute.nc</dataSource>
      </dataFile>
      <dataFile label="input" target="INPUT/" chksum="" size="" timestamp="">
        <dataSource platform="$(platform)">$(CMIP6_ARCHIVE_ROOT)/CM4/common/common_LM4/biodata.nc</dataSource>
      </dataFile>
      <dataFile label="input" target="INPUT/" chksum="" size="" timestamp="">
        <dataSource platform="$(platform)">$(CMIP6_ARCHIVE_ROOT)/CM4/common/common_LM4/cover_type.nc</dataSource>
      </dataFile>
      <dataFile label="input" target="INPUT/" chksum="" size="" timestamp="">
        <dataSource platform="$(platform)">$(CMIP6_ARCHIVE_ROOT)/CM4/common/common_LM4/geohydrology.nc</dataSource>
      </dataFile>
      <dataFile label="input" target="INPUT/" chksum="" size="" timestamp="">
        <dataSource platform="$(platform)">$(CMIP6_ARCHIVE_ROOT)/CM4/common/common_LM4/geohydrology_table_2a2n.nc</dataSource>
      </dataFile>
      <dataFile label="input" target="INPUT/" chksum="" size="" timestamp="">
        <dataSource platform="$(platform)">$(CMIP6_ARCHIVE_ROOT)/CM4/common/common_LM4/ground_type.nc</dataSource>
      </dataFile>
      <dataFile label="input" target="INPUT/" chksum="" size="" timestamp="">
        <dataSource platform="$(platform)">$(CMIP6_ARCHIVE_ROOT)/CM4/common/common_LM4/landuse.nc</dataSource>
      </dataFile>
      <dataFile label="input" target="INPUT/" chksum="" size="" timestamp="">
        <dataSource platform="$(platform)">$(CMIP6_ARCHIVE_ROOT)/CM4/common/common_LM4/soil_brdf.nc</dataSource>
      </dataFile>
    </input>
  </freInclude>
```

This block includes another possibly platform-dependent variable, `CMIP6_ARCHIVE_ROOT`, that we need to add as as `property` inside the `platform` block:

```xml
      <property name="CMIP6_ARCHIVE_ROOT"  value="$PDATA/gfdl/cmip6/datasets"/>
```

`c96_LM4` provides resolution-dependent input data for the land-model:

```xml
  <freInclude name="c96_LM4">
    <input>
      ...
    </input>
  </freInclude>
```

The lists of data files all follow a common format and don't use any new variables, so I'm omitting them.

The remaining input files provide information related to aerosols, chemistry, trace gases, volcanoes, emissions, and (in the final file) sea surface temperature. **TODO: figure out what these input files actually provide.**

```xml
  <freInclude name="common_AM4_v20170208">
    <input>
      ...
    </input>
  </freInclude>

  <freInclude name="common_cmip6">
    <input>
      ...
    </input>
  </freInclude>

  <freInclude name="Volcano_Solar_TimeSeries_cmip6">
    <input>
      ...
    </input>
  </freInclude>

  <freInclude name="emissions_cmip6_v20161213">
    <input>
      ...
    </input>
  </freInclude>

  <freInclude name="L32_gas_conc">
    <input>
      ...
    </input>
  </freInclude>

  <freInclude name="amip_sst_cmip6">
    <input>
      ...
    </input>
  </freInclude>
```

The next entries in the `input` block list input data tables:

```xml
      <xi:include xpointer="xpointer(//freInclude[@name='data_table_AM4']/input/node())"/>
      <xi:include xpointer="xpointer(//freInclude[@name='data_table_co2']/input/node())"/>
```

These also require `freInclude` entries that, in this case, reference files in `INPUT`:

```xml
  <freInclude name="data_table_AM4">
    <input>
      <dataFile label="dataTable" target="INPUT/" chksum="" size="" timestamp="">
        <dataSource platform="$(platform)">$(INPUT)/data_table/data_table_amip</dataSource>
      </dataFile>
    </input>
  </freInclude>

  <freInclude name="data_table_co2">
    <input>
      <dataFile label="dataTable" target="INPUT/" chksum="" size="" timestamp="">
        <dataSource platform="$(platform)">$(INPUT)/data_table/data_table_co2</dataSource>
      </dataFile>
    </input>
  </freInclude>
```

These data tables have to be added to `$(INPUT)/data_table`. As for namelists, I'm assuming you already know how to configure these files.

The next entry provides information about the model grid:

```xml
      <xi:include xpointer="xpointer(//freInclude[@name='c96_grid']/input/node())"/>
```

The corresponding `freInclude` is
```xml
  <freInclude name="c96_grid">
    <input>
      <dataFile label="gridSpec" target="INPUT/" chksum="" size="" timestamp="">
        <dataSource platform="$(platform)">$(CMIP6_ARCHIVE_ROOT)/CM4/common/c96_grid/c96_OM4_025_grid_No_mg_drag_v20160808.tar</dataSource>
      </dataFile>
      <dataFile label="input" target="INPUT/land_domain.nc" chksum="" size="" timestamp="">
        <dataSource platform="$(platform)">$(CMIP6_ARCHIVE_ROOT)/CM4/common/c96_grid/land_domain.nc</dataSource>
      </dataFile>
    </input>
  </freInclude>
```

The next two entries,

```xml
      <xi:include xpointer="xpointer(//freInclude[@name='c96_stack_size']/input/node())"/>
      <xi:include xpointer="xpointer(//freInclude[@name='csh_adjust_dry_mass']/input/node())"/>
```

are a little different: they provide CSH code that is injected directly into the run script:

```xml
  <freInclude name="c96_stack_size">
    <input>
      <csh><![CDATA[
set domains_stack_size = "2097152"

# Copy AWG ascii input files to GFDL (waiting for FRE to do this for us)
if ( $?batch && ! $?FRE_STAGE ) then
  if ( $?flagOutputXferOn && $?flagOutputPostProcessOn ) then
    gcp --batch -cd -r --sync $(INPUT) gfdl:$(INPUT_GFDL)/..
  endif
endif
      ]]></csh>
    </input>
  </freInclude>

  <freInclude name="csh_adjust_dry_mass">
    <input>
      <csh type="always"><![CDATA[
pushd INPUT
#cd INPUT

# adjustment of dry mass first time only
set adjust_dry_mass = `$(INPUT)/adjust_dry_mass.csh`

# create dummy MOM6 parameter file
touch MOM_input
popd
      ]]></csh>
    </input>
  </freInclude>

```

This section references a platform-independent variable, `INPUT_GFDL`, that provides a path in the GFDL home directories where a copy of small text-based input files will be stored. Because this variable's platform-independent, we'll define it as a top-level `property` (at the same level as e.g. `FRESTEM`):

```xml
  <property name="INPUT_GFDL" value="$home/$USER/ncrc/$(FRE_STEM)/input"/>
```

The section also references a CSH script (`adjust_dry_mass.csh`) that must be placed in `INPUT`. Like other ASCII input files, I'm assuming you already have a copy ready.

The next section references initial conditions:

```xml
      <xi:include xpointer="xpointer(//freInclude[@name='c96L33_initCond']/input/node())"/>
```

```xml
  <freInclude name="c96L33_initCond">
    <input>
      <dataFile label="initCond" target="INPUT/" chksum="" size="" timestamp="">
        <dataSource platform="$(platform)">/lustre/f2/pdata/gfdl/gfdl_W/Ming.Zhao/archive/mdt/awg/input/ic/AM4/awg_C96L33_LM3_OM4_025_grid_lm4p1_topo_v20170309.tar</dataSource>
      </dataFile>
    </input>
  </freInclude>
```

and the final section references two final ASCII inputs that must be placed in subdirectories of `INPUT`:

```xml
      <xi:include xpointer="xpointer(//freInclude[@name='diag_table_lm3p2_fullaero']/input/node())"/>
      <xi:include xpointer="xpointer(//freInclude[@name='field_table_am4p11']/input/node())"/>

    </input>
```

```xml
  <freInclude name="diag_table_lm3p2_fullaero">
    <input>
      <diagTable>
$name
$baseDate

      </diagTable>
      <dataFile label="diagTable" target="INPUT/" chksum="" size="" timestamp="">
        <dataSource platform="$(platform)">$(INPUT)/diag_table/diag_table_atmos</dataSource>
      </dataFile>
      <dataFile label="diagTable" target="INPUT/" chksum="" size="" timestamp="">
        <dataSource platform="$(platform)">$(INPUT)/diag_table/diag_table_atmos_simpleSO4</dataSource>
      </dataFile>
      <dataFile label="diagTable" target="INPUT/" chksum="" size="" timestamp="">
         <dataSource platform="$(platform)">$(INPUT)/diag_table/diag_table_land_old</dataSource>
      </dataFile>
    </input>
  </freInclude>

  <freInclude name="field_table_am4p11">
    <input>
      <dataFile label="fieldTable" target="INPUT/" chksum="" size="" timestamp="">
        <dataSource platform="$(platform)">$(INPUT)/field_table/field_table_am4p11</dataSource>
      </dataFile>
    </input>
  </freInclude>
```

Next, a `runtime` block contains information about the simulation length and parallelization scheme:

```xml
    <runtime>
      <production simTime="2" units="years" >
        <segment simTime="12" units="months" />
        <xi:include xpointer="xpointer(//freInclude[@name='c96_864_production']/node())"/>
      </production>
      <regression name="basic">
          <xi:include xpointer="xpointer(//freInclude[@name='c96_216_basic']/node())"/>
      </regression>
    </runtime>
```

Note that `production` configures the default "production" run, while named `regression` blocks can be used to run shorter tests. There can be multiple `regression` blocks, and they are selected based on the `name` attribute by the `-r` `frerun` flag. The referenced `freInclude` blocks provides information about resource requirements and domain decomposition for different model components:

```xml
  <freInclude name="c96_864_production">
    <resources jobWallclock="5:00:00"
               segRuntime="2:00:00" >
      <atm ranks="432" threads="2"    layout = "3,24"    io_layout = "1,4" />
      <lnd                            layout = "3,24"    io_layout = "1,4" />
      <ocn ranks="0"   threads="0"    layout = "144,3"   io_layout = "1,3" />
      <ice                            layout = "144,3"   io_layout = "1,3" />
    </resources>
  </freInclude>

  <freInclude name="c96_216_basic">
    <run days="8">
      <resources jobWallclock="00:40:00">
        <atm ranks="216" threads="1"    layout = "3,12"    io_layout = "1,4" />
        <lnd                            layout = "3,12"    io_layout = "1,4" />
        <ocn ranks="0"   threads="0"    layout = "72,3"    io_layout = "1,3" />
        <ice                            layout = "72,3"    io_layout = "1,3" />
      </resources>
    </run>
  </freInclude>
```

Finally, a `postprocess` block can be used to configure postprocessing tasks, but we're leaving this empty for now.

```xml
    <postProcess>
    </postProcess>
  </experiment>
```

After the run completes, files will be copied to GFDL storage, and `frerun` expects a `platform` block to determine appropriate paths. This platform must have a `name` attribute that is the same as the platform used to run the model, but with `gfdl` prefixed to it. So, inside the `setup` block, add

```xml
    <platform name="gfdl.ncrc4-intel19">
      <freVersion>$(FRE_VERSION)</freVersion>
      <directory stem="$(FRE_STEM)">
        <scripts>/home/$USER/$(stem)/$(FMS_COMP_EXP)/$(name)/$(platform)-$(target)/scripts</scripts>
        <stdout>/home/$USER/$(stem)/$(FMS_COMP_EXP)/$(name)/$(platform)-$(target)/stdout</stdout>
        <state>/home/$USER/$(stem)/$(FMS_COMP_EXP)/$(name)/$(platform)-$(target)/state</state>
        <archive>$ARCHIVE/$(stem)/$(FMS_COMP_EXP)/$(name)/$(platform)-$(target)</archive>
        <postProcess>$ARCHIVE/$(stem)/$(FMS_COMP_EXP)/$(name)/$(platform)-$(target)/pp</postProcess>
        <analysis>/nbhome/$USER/$(stem)/$(FMS_COMP_EXP)/$(name)</analysis>
      </directory>
      <property name="F2003_FLAGS"         value="-DINTERNAL_FILE_NML "/>
      <property name="INPUT"               value="$(INPUT_GFDL)"/>
      <property name="CMIP6_ARCHIVE_ROOT"  value="/archive/oar.gfdl.cmip6/datasets"/>
      <property name="NCORE_PER_NODE"      value="1"/>
      <csh><![CDATA[
        module load git
        ]]>
      </csh>
    </platform>
```

To run the regression experiment with `name="basic"`, invoke

```console
$ frerun -x default.xml -p ncrc4.intel19 -t openmp -r basic -q normal c96L33_am4p0
```

The `-x`, `-p`, and `-t` flags are the same as for `fremake`. The `-r` flag selects the regression test, and the `-q` flag controls the quality-of-service string passed to the scheduler when the run is submitted. The final argument identifies the name of the experiment in the XML file. This should produce a run script in the `scripts` directory. To run the experiment, submit the script to the scheduler:

```console
$ sbatch $HOME/am4/xanadu/scripts/ncrc4.intel19-prod-openmp/run/c96L33_am4p0_donner_1x0m8d_216x1a
```
