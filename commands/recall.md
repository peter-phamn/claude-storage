Search ICM memory for: $ARGUMENTS

Run:
```bash
if [ -z "$ARGUMENTS" ]; then
  icm wake-up --max-tokens 800
else
  icm recall "$ARGUMENTS" --limit 10
fi
```
