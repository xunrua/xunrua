# Theme Management

This repository maintains multiple themed versions of the GitHub profile README.

## Themes

### Genshin Impact - Raiden Shogun

- **Tag**: `theme/genshin-raiden`
- **Primary**: Purple (#9b6dff)
- **Character**: Raiden Shogun (雷电将军)
- **Location**: `themes/genshin/`
- **Backup commit**: The original theme is tagged via git.

### Manchester City FC

- **Tag**: `theme/mancity`
- **Primary**: Sky Blue (#6CABDD)
- **Secondary**: Navy (#1C2C5B)
- **Accent**: Gold (#FFCE00)
- **Character**: Erling Haaland (埃尔林·哈兰德)
- **Location**: `themes/mancity/`

## Switching Themes

To switch between themes:

```bash
# Switch to Genshin theme
git checkout theme/genshin

# Switch to Manchester City theme
git checkout theme/mancity

# Or restore from tag
git checkout theme/genshin-raiden -- .
git add -A
git commit -m "chore: switch to genshin theme"
```

## Directory Structure

```
themes/
  genshin/
    README.md          # Original Genshin README
    assets/            # Element icons, Raiden portrait, typing SVG
  mancity/
    README.md          # Manchester City README
    assets/            # Crest, Haaland photo, typing SVG
```

The root `README.md` and `assets/` directory always reflect the currently active theme.
