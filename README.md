## The branch to manage the astro-paper deploy

### Useful commands
```
npm run dev
npm run build
npm run preview
```

### Clean build
```
Remove-Item -Recurse -Force node_modules
Remove-Item package-lock.json
npm install
npm run build
```

### Init another astro build for testing
```
npm create astro@latest -- --template hsufit/astro-paper-hsufit
npm run build
```

### should run before powershell run npm command
```
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope Process
```