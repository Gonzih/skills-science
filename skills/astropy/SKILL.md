---
name: astropy
description: Astronomical coordinates, units, FITS I/O, cosmology calculations, and Vizier/SIMBAD queries with Astropy
license: MIT
triggers:
  - astropy
  - astronomical coordinates
  - convert ra dec
  - fits file astronomy
  - cosmological distance
  - simbad lookup
  - vizier catalog
  - galactic coordinates
  - redshift distance
  - julian date
metadata:
  skill-author: gonzih
  data-sources:
    - vizier.cds.unistra.fr — 20K+ astronomical catalogs (2MASS, Gaia, SDSS etc); free; no auth
    - simbad.cds.unistra.fr — object identifiers, coordinates, measurements; free; no auth
    - fits files — any local or remote FITS data file
  byok:
    - none
  compatibility: claude-code>=1.0
---

# Astropy Skill

## What it does
Uses Astropy for astronomy calculations: coordinate transforms (ICRS, FK5, Galactic, AltAz), unit conversions, FITS file reading/writing, cosmological distance calculations (luminosity distance, comoving distance, age of universe), time conversions (UTC, JD, MJD, ISOT), and remote catalog queries via astroquery (Vizier, SIMBAD).

## How to invoke
- "Convert RA=83.82, Dec=−5.39 to Galactic coordinates"
- "Calculate the luminosity distance at redshift z=0.5 (Planck18 cosmology)"
- "Query SIMBAD for object M31"
- "Read a FITS file and show its header"
- "What is the age of the universe in Planck18 cosmology?"
- "Convert 2025-01-15T12:00:00 to Julian Date"
- "Query Vizier for 2MASS sources around this position"

## Key parameters
- `ra, dec` — right ascension / declination in degrees
- `frame` — coordinate frame: `icrs`, `fk5`, `galactic`, `altaz`
- `z` — cosmological redshift
- `cosmology` — `Planck18`, `WMAP9`, `FlatLambdaCDM` custom
- `fits_path` — path to local FITS file
- `radius` — search radius (astropy Angle or string like `'5 arcmin'`)

## Rate limits
- Vizier/SIMBAD (via astroquery): polite use, ~1 req/s recommended
- No API key required

## Workflow steps
1. Install: `pip install astropy astroquery`
2. Create `SkyCoord` objects for coordinate work
3. Use `astropy.cosmology` for distance/age calculations
4. Use `astropy.io.fits` for FITS file operations
5. Use `astroquery.vizier` / `astroquery.simbad` for catalog queries

## Working code example

```python
import numpy as np
from astropy import units as u
from astropy.coordinates import SkyCoord, Galactic, AltAz, EarthLocation
from astropy.cosmology import Planck18, WMAP9, FlatLambdaCDM
from astropy.time import Time
from astropy.io import fits
from astropy.table import Table

# astroquery for remote catalog access
try:
    from astroquery.vizier import Vizier
    from astroquery.simbad import Simbad
    ASTROQUERY_AVAILABLE = True
except ImportError:
    ASTROQUERY_AVAILABLE = False
    print("Install astroquery: pip install astroquery")


# ── Coordinates ───────────────────────────────────────────────────────────────

def icrs_to_galactic(ra_deg: float, dec_deg: float) -> dict:
    """Convert ICRS (RA/Dec) to Galactic coordinates."""
    c = SkyCoord(ra=ra_deg * u.deg, dec=dec_deg * u.deg, frame="icrs")
    gal = c.galactic
    return {
        "ra_deg": ra_deg,
        "dec_deg": dec_deg,
        "l_deg": round(gal.l.deg, 6),
        "b_deg": round(gal.b.deg, 6),
        "ra_hms": c.ra.to_string(unit=u.hour, sep="hms", precision=2),
        "dec_dms": c.dec.to_string(sep="dms", precision=1),
    }


def angular_separation(ra1: float, dec1: float, ra2: float, dec2: float) -> dict:
    """Angular separation between two sky positions."""
    c1 = SkyCoord(ra=ra1 * u.deg, dec=dec1 * u.deg, frame="icrs")
    c2 = SkyCoord(ra=ra2 * u.deg, dec=dec2 * u.deg, frame="icrs")
    sep = c1.separation(c2)
    return {
        "degrees": round(sep.deg, 6),
        "arcmin": round(sep.arcmin, 4),
        "arcsec": round(sep.arcsec, 3),
    }


def altaz_from_icrs(ra_deg: float, dec_deg: float,
                    lat_deg: float, lon_deg: float, height_m: float,
                    time_utc: str) -> dict:
    """Convert ICRS coordinates to Alt/Az for an observer and time."""
    location = EarthLocation(lat=lat_deg * u.deg, lon=lon_deg * u.deg, height=height_m * u.m)
    obs_time = Time(time_utc, format="isot", scale="utc")
    altaz_frame = AltAz(obstime=obs_time, location=location)
    c = SkyCoord(ra=ra_deg * u.deg, dec=dec_deg * u.deg, frame="icrs")
    altaz = c.transform_to(altaz_frame)
    return {
        "altitude_deg": round(altaz.alt.deg, 4),
        "azimuth_deg": round(altaz.az.deg, 4),
        "airmass": round(float(altaz.secz), 3) if altaz.alt.deg > 0 else None,
        "above_horizon": altaz.alt.deg > 0,
    }


# ── Cosmology ─────────────────────────────────────────────────────────────────

def cosmological_distances(z: float, cosmology_name: str = "Planck18") -> dict:
    """
    Compute cosmological distances at redshift z.
    cosmology_name: 'Planck18', 'WMAP9'
    """
    cosmo = {"Planck18": Planck18, "WMAP9": WMAP9}.get(cosmology_name, Planck18)
    return {
        "redshift": z,
        "cosmology": cosmology_name,
        "H0": float(cosmo.H0.value),
        "Om0": cosmo.Om0,
        "comoving_distance_Mpc": round(cosmo.comoving_distance(z).to(u.Mpc).value, 3),
        "luminosity_distance_Mpc": round(cosmo.luminosity_distance(z).to(u.Mpc).value, 3),
        "angular_diameter_distance_Mpc": round(cosmo.angular_diameter_distance(z).to(u.Mpc).value, 3),
        "lookback_time_Gyr": round(cosmo.lookback_time(z).to(u.Gyr).value, 4),
        "age_at_z_Gyr": round(cosmo.age(z).to(u.Gyr).value, 4),
    }


def universe_age(cosmology_name: str = "Planck18") -> dict:
    """Current age and key parameters of the universe."""
    cosmo = {"Planck18": Planck18, "WMAP9": WMAP9}.get(cosmology_name, Planck18)
    return {
        "cosmology": cosmology_name,
        "age_Gyr": round(cosmo.age(0).to(u.Gyr).value, 4),
        "H0_km_s_Mpc": float(cosmo.H0.value),
        "Om0": cosmo.Om0,
        "Ode0": cosmo.Ode0,
        "Tcmb0_K": float(cosmo.Tcmb0.value),
        "critical_density_g_cm3": float(cosmo.critical_density(0).to(u.g / u.cm**3).value),
    }


def custom_cosmology(H0: float, Om0: float, z: float) -> dict:
    """Compute distances for a custom flat ΛCDM cosmology."""
    cosmo = FlatLambdaCDM(H0=H0, Om0=Om0)
    return {
        "H0": H0,
        "Om0": Om0,
        "redshift": z,
        "luminosity_distance_Mpc": round(cosmo.luminosity_distance(z).to(u.Mpc).value, 3),
        "comoving_distance_Mpc": round(cosmo.comoving_distance(z).to(u.Mpc).value, 3),
        "lookback_time_Gyr": round(cosmo.lookback_time(z).to(u.Gyr).value, 4),
    }


# ── Time conversion ───────────────────────────────────────────────────────────

def convert_time(time_str: str, in_format: str = "isot") -> dict:
    """
    Convert between time formats.
    in_format: 'isot' (ISO 8601), 'jd', 'mjd', 'unix', 'fits'
    """
    t = Time(time_str, format=in_format, scale="utc")
    return {
        "utc": t.isot,
        "jd": round(t.jd, 8),
        "mjd": round(t.mjd, 8),
        "unix": round(t.unix, 3),
        "gps": round(t.gps, 3),
        "fits": t.fits,
    }


# ── FITS I/O ──────────────────────────────────────────────────────────────────

def read_fits_header(filepath: str, hdu_index: int = 0) -> dict:
    """Read and return FITS header as a dict."""
    with fits.open(filepath) as hdul:
        hdr = hdul[hdu_index].header
        data_shape = hdul[hdu_index].data.shape if hdul[hdu_index].data is not None else None
        return {
            "n_hdus": len(hdul),
            "hdu_index": hdu_index,
            "data_shape": data_shape,
            "header": dict(hdr),
        }


def fits_table_to_list(filepath: str, hdu_index: int = 1) -> list[dict]:
    """Read a FITS binary table as a list of row dicts."""
    with fits.open(filepath) as hdul:
        tbl = Table(hdul[hdu_index].data)
    return [dict(zip(tbl.colnames, row)) for row in tbl]


# ── Catalog queries (astroquery) ──────────────────────────────────────────────

def simbad_query_object(name: str) -> dict:
    """Query SIMBAD for a named object."""
    if not ASTROQUERY_AVAILABLE:
        raise ImportError("astroquery required: pip install astroquery")
    simbad = Simbad()
    simbad.add_votable_fields("ra", "dec", "otype", "distance", "z_value", "flux(V)")
    result = simbad.query_object(name)
    if result is None:
        return {"error": f"Object '{name}' not found in SIMBAD"}
    row = result[0]
    return {
        "name": name,
        "ra": str(row["RA"]),
        "dec": str(row["DEC"]),
        "object_type": str(row["OTYPE"]),
    }


def vizier_query_region(ra_deg: float, dec_deg: float, radius_arcmin: float = 5.0,
                        catalog: str = "II/246") -> list[dict]:
    """
    Query a Vizier catalog around a sky position.
    catalog: 'II/246' = 2MASS, 'I/350/gaiaedr3' = Gaia EDR3, 'V/147/sdss12' = SDSS DR12
    """
    if not ASTROQUERY_AVAILABLE:
        raise ImportError("astroquery required: pip install astroquery")
    v = Vizier(columns=["*"], row_limit=100)
    coord = SkyCoord(ra=ra_deg * u.deg, dec=dec_deg * u.deg, frame="icrs")
    result = v.query_region(coord, radius=radius_arcmin * u.arcmin, catalog=catalog)
    if not result:
        return []
    tbl = result[0]
    return [dict(zip(tbl.colnames, [str(v) for v in row])) for row in tbl[:50]]


# --- Example usage ---
if __name__ == "__main__":
    # Orion Nebula: RA=83.8221, Dec=-5.3911
    coords = icrs_to_galactic(83.8221, -5.3911)
    print(f"Orion Nebula — Galactic: l={coords['l_deg']:.3f}°, b={coords['b_deg']:.3f}°")

    sep = angular_separation(83.8221, -5.3911, 84.0, -5.5)
    print(f"Separation: {sep['arcmin']:.2f} arcmin")

    # Planck18 cosmology at z=1
    dist = cosmological_distances(1.0, "Planck18")
    print(f"\nAt z=1.0 (Planck18):")
    print(f"  Luminosity distance: {dist['luminosity_distance_Mpc']:.1f} Mpc")
    print(f"  Lookback time: {dist['lookback_time_Gyr']:.2f} Gyr")

    age = universe_age("Planck18")
    print(f"\nUniverse age: {age['age_Gyr']:.3f} Gyr (H0={age['H0_km_s_Mpc']} km/s/Mpc)")

    # Time conversion
    t = convert_time("2024-01-15T12:00:00", "isot")
    print(f"\n2024-01-15T12:00:00 → JD={t['jd']}, MJD={t['mjd']}")

    # SIMBAD query
    if ASTROQUERY_AVAILABLE:
        obj = simbad_query_object("M31")
        print(f"\nSIMBAD M31: RA={obj['ra']}, Dec={obj['dec']}, Type={obj['object_type']}")
```

## Live Data Sources
- **Vizier**: `https://vizier.cds.unistra.fr/` — 20K+ catalogs (2MASS, Gaia, SDSS, WISE, etc.)
- **SIMBAD**: `https://simbad.cds.unistra.fr/` — stellar/galaxy object database
- **Astropy documentation**: https://docs.astropy.org/
- **astroquery documentation**: https://astroquery.readthedocs.io/
- **Cosmology parameters**: Planck18 (H0=67.66, Om0=0.30966), WMAP9 (H0=69.32, Om0=0.2865)
