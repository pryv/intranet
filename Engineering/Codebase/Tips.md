# Tips

## VS Code

## js 'types' can only be used in a .ts file

Open the settings.json

Depending on your platform, the user settings file is located here:
- Windows %APPDATA%\Code\User\settings.json
- macOS $HOME/Library/Application Support/Code/User/settings.json
- Linux $HOME/.config/Code/User/settings.json

add the following line : `"javascript.validate.enable": false`

save

enjoy

Source : [stackoverflow ('best' answer is wrong)](https://stackoverflow.com/questions/48859169/js-types-can-only-be-used-in-a-ts-file-visual-studio-code-using-ts-check)