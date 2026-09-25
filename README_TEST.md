# MPAS Latent Heating Modification Tool

**Author:** May Wong & Alexander Lojko
**Last updated:** September 2026

The MPAS Latent Heating Modification Tool involves a set of configurations for the Model for Prediction Across Scales (MPAS) that enables controlled modification of cloud-related latent heating in simulations.

The tool allows users to amplify or suppress latent heating associated with parameterized cumulus convection and microphysics schemes through a set of configurable parameters. These modifications enable sensitivity experiments to investigate science questions related to cloud heating. 

The tool is designed to be used alongside the [`MPAS-Limited-Area-Masks`](https://github.com/maywswong/MPAS-Limited-Area-Masks) tool, which allows users to prescribe the geographic region over which cloud heating modifications are applied.

This README describes the required setup, configuration options, and recommended usage of the latent heating modification tool, including current limitations and caveats.

## Contents

* [Overview](#overview)
* [Setup](#setup)
* [Configuration](#configuration)
* [Vertical Weighting Function](#vertical-weighting-function)
* [Example Configurations](#example-configurations)
* [Caveats and Additional Notes](#caveats-and-additional-notes)
* [Reporting Bugs](#reporting-bugs)
* [Release Notes](#release-notes)

---

## Overview

The latent heating modification tool provides the following capabilities:

* **Geographic control:** Apply latent heating modifications within a user-defined region using an MPAS NetCDF mask.
* **Configurable heating perturbations:** Amplify or suppress cloud-related latent heating using a single scaling parameter.
* **Vertical control:** Prescribe the vertical structure of the heating modifications using a cosine weighting function.
  
The modifications are controlled through `namelist.atmosphere`, allowing users to configure experiments without recompiling MPAS each time. 

**Current limitations:** Microphysics heating modifications are currently implemented for the Thompson microphysics scheme only. Cumulus heating modifications are supported for all cumulus parameterization schemes.

## Setup

### 1. Compile the modified MPAS model

Compile this version of the MPAS model containing source code modifications to modify cloud heating. Once compiled, the necessary files will be created to enable configuration of the cloud heating experiments. 

### 2. Create a geographic mask

Before running a simulation, a geographic mask must be generated for the region over which the latent heating modifications will be applied.

The mask is generated using the `create_mask` functionality from the [MPAS-Limited-Area-Masks](https://github.com/maywswong/MPAS-Limited-Area-Masks/blob/create_mask/README.md) repository.

Follow the instructions in that repository to define the desired geographic region and generate the corresponding NetCDF mask file.

### 3. Configure the mask in `streams.atmosphere`

Once the mask file has been generated, specify the filename you have attributed to the mask in the `nudge_mask` input stream in `streams.atmosphere`.

For example, the following stream configuration specifies a mask file named `x1.40962.mask_region.nc`:

```xml
<immutable_stream name="nudge_mask"
                 type="input"
                 filename_template="x1.40962.mask_region.nc"
                 packages="nudging"
                 input_interval="initial_only" />
```

Replace the `filename_template` with the name of your generated mask file.

The mask is read during model initialization and is used to determine the geographic region in which the latent heating modifications are applied.

**Note:** Make sure the mask is placed (or linked) within your compiled MPAS work-space. 

## Configuration

The latent heating modification tool is configured through the `&nudging` namelist in `namelist.atmosphere`.

The parameters described below provide control for the user's intended cloud heating modifications.

### Default configuration

The default configuration is:

```fortran
&nudging
 config_nudge_mask = 'off'
 config_mod_mp_heating = 'off'
 config_mod_cu_heating = 'off'
 config_const_nudge_fac = 0
 config_z0 = 1500
 config_z1 = 2500
 config_nudge_bottom = 0
/
```

By default, all latent heating modifications are disabled. The functionality requires nudging to be enabled and a valid geographic mask to be available.

### Configuration parameters

| Parameter                | Description                                                                | Default |
| :----------------------- | :------------------------------------------------------------------------- | :------ |
| `config_nudge_mask`      | Enables the geographic mask required for the latent heating modifications. | `'off'` |
| `config_mod_mp_heating`  | Enables modification of microphysics-related latent heating.               | `'off'` |
| `config_mod_cu_heating`  | Enables modification of cumulus-related latent heating.                    | `'off'` |
| `config_const_nudge_fac` | Controls the magnitude and sign of the latent heating modification.        | `0`     |
| `config_z0`              | Lower boundary of the vertical weighting function, in meters.              | `1500`  |
| `config_z1`              | Upper boundary of the vertical weighting function, in meters.              | `2500`  |
| `config_nudge_bottom`    | Controls the vertical orientation of the weighting function.               | `0`     |

### `config_nudge_mask`

Enables or disables the geographic mask.

* `'on'`: Enables the mask and allows heating modifications to be restricted to the prescribed geographic region.
* `'off'`: Disables the mask.

**Important:** The current implementation requires the mask to be enabled for the latent heating modification tool to operate as intended.

### `config_mod_mp_heating`

Enables or disables modifications to microphysics-related latent heating.

* `'on'`: Applies the prescribed heating modification to the microphysics heating tendencies.
* `'off'`: Leaves microphysics heating unmodified.

Currently, this option is implemented for the Thompson microphysics scheme only.

### `config_mod_cu_heating`

Enables or disables modifications to cumulus-related latent heating.

* `'on'`: Applies the prescribed heating modification to the cumulus heating tendencies.
* `'off'`: Leaves cumulus heating unmodified.

This option is supported for all cumulus parameterization schemes.

### `config_const_nudge_fac`

Controls the magnitude and sign of the latent heating modification.

The parameter determines the fractional change in latent heating, subject to the geographic mask and vertical weighting function.

|  Value | Effect on latent heating                                              |
| :----: | :-------------------------------------------------------------------- |
|  `-1`  | 100% suppression of latent heating (some residual heating remains).   |
| `-0.5` | 50% suppression of latent heating.                                    |
|   `0`  | No modification to latent heating.                                    |
|  `0.5` | 50% enhancement of latent heating.                                    |
|   `1`  | 100% enhancement of latent heating (doubling of heating).      |

Positive values enhance heating, while negative values suppress heating.

The modification has been tested primarily for values between `-1` and `1`. Values outside this range have not been systematically tested and should be used with caution.

The actual heating modification at a given grid point depends on the specified geographic mask and vertical weighting function.

### `config_z0` and `config_z1`

These parameters define the vertical transition region of the heating modification.

* `config_z0`: Lower boundary of the vertical weighting function (meters).
* `config_z1`: Upper boundary of the vertical weighting function (meters).

The cosine weighting function smoothly transitions between regions of maximum heating modification and regions where no modification is applied.

The exact vertical structure depends on `config_nudge_bottom`, as described in the [Vertical Weighting Function](#vertical-weighting-function) section.

### `config_nudge_bottom`

Controls whether the maximum heating modification is applied above or below the vertical transition region.

| Value | Vertical configuration                                                             |
| :---: | :--------------------------------------------------------------------------------- |
|  `0`  | Maximum heating modification above `config_z1`; no modification below `config_z0`. |
|  `1`  | Maximum heating modification below `config_z0`; no modification above `config_z1`. |

The region between `config_z0` and `config_z1` acts as a transition zone, where the heating modification is smoothly tapered using a cosine weighting function.

## Vertical Weighting Function

The vertical weighting function allows users to prescribe the vertical structure of the latent heating modifications.

The weighting function is controlled by `config_z0`, `config_z1`, and `config_nudge_bottom`. Within the geographic mask, the prescribed heating modification is scaled by a cosine function between the lower and upper boundaries.

### 1. Upper-level heating modifications

Set:

```fortran
config_nudge_bottom = 0
```

In this configuration:

* Below `config_z0`, no latent heating modification is applied.
* Between `config_z0` and `config_z1`, the heating modification smoothly increases according to a cosine weighting function.
* Above `config_z1`, the full modification prescribed by `config_const_nudge_fac` is applied.

This configuration is useful for experiments targeting upper-level cloud heating while minimizing modifications to lower-level heating.

### 2. Lower-level heating modifications

Set:

```fortran
config_nudge_bottom = 1
```

In this configuration:

* Below `config_z0`, the full heating modification prescribed by `config_const_nudge_fac` is applied.
* Between `config_z0` and `config_z1`, the heating modification smoothly decreases according to a cosine weighting function.
* Above `config_z1`, no latent heating modification is applied.

This configuration is useful for experiments targeting lower-level cloud heating while minimizing modifications to upper-level heating.

### 3. Full-column heating modifications

To apply the heating modification throughout the atmospheric column, set:

```fortran
config_z1 = -1000
```

The value of `config_z0` can be set to any value.

The choice of `-1000 m` ensures that the upper boundary of the weighting function is below the minimum model height (`zm`) encountered in the current implementation, including locations with negative terrain-following heights, such as over the Caspian Sea.

This configuration should be used when the intention is to apply a vertically uniform heating modification throughout the model column.

## Example Configurations

The following examples illustrate how the namelist parameters can be combined to define different heating modification experiments.

### Example 1: Suppress microphysics heating throughout the column

The following configuration suppresses Thompson microphysics heating by 50% throughout the geographic mask.

```fortran
&nudging
 config_nudge_mask = 'on'
 config_mod_mp_heating = 'on'
 config_mod_cu_heating = 'off'
 config_const_nudge_fac = -0.5
 config_z0 = 1500
 config_z1 = -1000
 config_nudge_bottom = 0
/
```

### Example 2: Enhance cumulus and microphysics heating at upper levels

The following configuration enhances cumulus and microphysics heating by 50% above 2500 m, with a cosine transition between 1500 and 2500 m.

```fortran
&nudging
 config_nudge_mask = 'on'
 config_mod_mp_heating = 'on'
 config_mod_cu_heating = 'on'
 config_const_nudge_fac = 0.5
 config_z0 = 1500
 config_z1 = 2500
 config_nudge_bottom = 0
/
```

These examples assume that the required geographic mask is specified in `streams.atmosphere` and that the modified MPAS executable has been compiled.

## Caveats and Additional Notes

Although the tool has undergone testing, some configurations and parameter values have not been systematically evaluated. So consider doing your own tests!

### Modifying Cumulus or Microphysics heating?

By default, if the user wants to modify cloud heating, we recommend enabling modification to both the microphysics and the cumulus scheme. 

### Microphysics scheme compatibility

Currently, microphysics heating modifications are supported only for the Thompson microphysics scheme.

This limitation arises because microphysics heating is treated separately for each microphysics scheme in the MPAS source code. In contrast, the cumulus heating modification functionality is implemented in a manner that supports all cumulus parameterization schemes.

### Vertical domain limits

If both `config_z0` and `config_z1` are greater than or equal to the model-top height (`ztop`), the heating modifications will also be turned off.

## Reporting Bugs

If you encounter a bug or unexpected behavior when using the latent heating modification tool, please report it through the [Issues page](../../issues) of this GitHub repository.

When reporting an issue, please include:

* The MPAS version and model configuration.
* The microphysics and cumulus schemes being used.
* The relevant `namelist.atmosphere` settings.
* A description of the unexpected behavior and, where possible, relevant model output or error messages.

## Release Notes

### 1.0

* Initial working version of the MPAS latent heating modification tool.
