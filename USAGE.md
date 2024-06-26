# Usage Guide

## The easy way

The easist way to use these packages is by creating a project with `Briefcase
<https://github.com/beeware/briefcase>`__. Briefcase will download pre-compiled
versions of these support packages, and add them to an Xcode project (or
pre-build stub application, in the case of macOS).

## The manual way

The Python support package *can* be manually added to any Xcode project;
however, you'll need to perform some steps manually (essentially reproducing what
Briefcase is doing)

**NOTE** Briefcase usage is the officially supported approach for using this
support package. If you are experiencing diffculties, one approach for debugging
is to generate a "Hello World" project with Briefcase, and compare the project that
Briefcase has generated with your own project.

To add this support package to your own project:

1. [Download a release tarball for your desired Python version and Apple
   platform](https://github.com/beeware/Python-Apple-support/releases)

2. Add the `python-stdlib` and `Python.xcframework` to your Xcode project. Both
   the `python-stdlib` folder and the `Python.xcframework` should be members of
   any target that needs to use Python.

3. In Xcode, select the root node of the project tree, and select the target you
   want to build.

4. Select "General" -> "Frameworks, Libraries and Embedded Content", and ensure
   that `Python.xcframework` is on the list of frameworks. It should be marked
   "Do not embed".

5. Select "General" -> "Build Phases", and ensure that the `python-stdlib` folder
   is listed in the "Copy Bundle Resources" step.

6. If you're on iOS, Add a new "Run script" build phase named "Purge Python Binary
   Modules for Non-Target Platforms". This script will purge any dynamic module for the
   platform you are *not* targeting. The script should have the following content:

```bash
if [ "$EFFECTIVE_PLATFORM_NAME" = "-iphonesimulator" ]; then
    echo "Purging Python modules for iOS Device"
    find "$CODESIGNING_FOLDER_PATH/python-stdlib" -name "*.*-iphoneos.dylib" -exec rm -f "{}" \;
else
    echo "Purging Python modules for iOS Simulator"
    find "$CODESIGNING_FOLDER_PATH" -name "*.*-iphonesimulator.dylib" -exec rm -f "{}" \;
fi
```

7. Add a new "Run script" build phase named "Sign Python Binary Modules".

   The iOS App Store requires that binary modules *must* be contained inside frameworks.
   This script will move every `.dylib` file in the `lib-dynload` folder to a unique
   framework in the `Frameworks` folder of your packaged binary, then sign the new
   framework. The script should have the following content:

```bash
set -e

install_dylib () {
    INSTALL_BASE=$1
    FULL_DYLIB=$2

    # The name of the .dylib file
    DYLIB=$(basename "$FULL_DYLIB")
    # The name of the .dylib file, relative to the install base
    RELATIVE_DYLIB=${FULL_DYLIB#$CODESIGNING_FOLDER_PATH/$INSTALL_BASE/}
    # The (hopefully unique) name of the framework, constructed by replacing path
    # separators in the relative name with underscores.
    FRAMEWORK_NAME=$(echo $RELATIVE_DYLIB | cut -d "." -f 1 | tr "/" "_");
    # A bundle identifier; not actually used, but required by Xcode framework packaging
    FRAMEWORK_BUNDLE_ID=$(echo $PRODUCT_BUNDLE_IDENTIFIER.$FRAMEWORK_NAME | tr "_" "-")
    # The name of the framework folder.
    FRAMEWORK_FOLDER="Frameworks/$FRAMEWORK_NAME.framework"

    # If the framework folder doesn't exist, create it.
    if [ ! -d "$CODESIGNING_FOLDER_PATH/$FRAMEWORK_FOLDER" ]; then
        echo "Creating framework for $RELATIVE_DYLIB"
        mkdir -p "$CODESIGNING_FOLDER_PATH/$FRAMEWORK_FOLDER"

        cp "$CODESIGNING_FOLDER_PATH/dylib-Info-template.plist" "$CODESIGNING_FOLDER_PATH/$FRAMEWORK_FOLDER/Info.plist"
        defaults write "$CODESIGNING_FOLDER_PATH/$FRAMEWORK_FOLDER/Info.plist" CFBundleExecutable -string "$DYLIB"
        defaults write "$CODESIGNING_FOLDER_PATH/$FRAMEWORK_FOLDER/Info.plist" CFBundleIdentifier -string "$FRAMEWORK_BUNDLE_ID"
    fi

    echo "Installing binary for $RELATIVE_DYLIB"
    mv "$FULL_DYLIB" "$CODESIGNING_FOLDER_PATH/$FRAMEWORK_FOLDER"
}

echo "Install standard library dylibs..."
find "$CODESIGNING_FOLDER_PATH/python-stdlib/lib-dynload" -name "*.dylib" | while read FULL_DYLIB; do
    install_dylib python-stdlib/lib-dynload "$FULL_DYLIB"
done

# Clean up dylib template
rm -f "$CODESIGNING_FOLDER_PATH/dylib-Info-template.plist"

echo "Signing frameworks as $EXPANDED_CODE_SIGN_IDENTITY_NAME ($EXPANDED_CODE_SIGN_IDENTITY)..."
find "$CODESIGNING_FOLDER_PATH/Frameworks" -name "*.framework" -exec /usr/bin/codesign --force --sign "$EXPANDED_CODE_SIGN_IDENTITY" ${OTHER_CODE_SIGN_FLAGS:-} -o runtime --timestamp=none --preserve-metadata=identifier,entitlements,flags --generate-entitlement-der "{}" \;
```

   You'll also need to add a file named `dylib-Info-template.plist` to your Xcode
   project, and make it a member of any target that needs to use Python. The template
   should have the following content:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>CFBundleDevelopmentRegion</key>
	<string>en</string>
	<key>CFBundleExecutable</key>
	<string></string>
	<key>CFBundleIdentifier</key>
	<string></string>
	<key>CFBundleInfoDictionaryVersion</key>
	<string>6.0</string>
	<key>CFBundlePackageType</key>
	<string>APPL</string>
	<key>CFBundleShortVersionString</key>
	<string>1.0</string>
	<key>CFBundleSupportedPlatforms</key>
	<array>
		<string>iPhoneOS</string>
	</array>
	<key>MinimumOSVersion</key>
	<string>12.0</string>
	<key>CFBundleVersion</key>
	<string>1</string>
</dict>
</plist>
```

   macOS projects don't require `.dylib` files be moved like this, so you can use a much
   simpler signing script:

```bash
set -e
echo "Signing as $EXPANDED_CODE_SIGN_IDENTITY_NAME ($EXPANDED_CODE_SIGN_IDENTITY)"
find "$CODESIGNING_FOLDER_PATH/Contents/Resources/python-stdlib/lib-dynload" -name "*.so" -exec /usr/bin/codesign --force --sign "$EXPANDED_CODE_SIGN_IDENTITY" -o runtime --timestamp=none --preserve-metadata=identifier,entitlements,flags --generate-entitlement-der {} \;
```

8. Disable dead code removal. As of March 2024 Xcode has "Dead Code Stripping" enabled by default. This _will_ remove functions from linked Python library that linker found unused.
   Navigate to Build Settings -> Linkin - General. Find "Dead Code Stripping" and select "No"

You will now be able to access the Python runtime in your Python code.

If you are on iOS, you will be able to deploy to an iOS simulator without specifying
development team; however, you will need to specify a valid development team to sign
the binaries for deployment onto a physical device (or for submission to the App Store).

If you are on macOS, you will need to specify a valid development team to run
the app. If you don't want to specify a development team in your project, you
will also need to enable the "Disable Library Validation" entitlement under
"Signing & Capabilities" -> "Hardened Runtime" for your project.

If you have any third party dependencies with binary components, they'll also need to go
through the processing of the scripts in steps 6 and 7.

## Accessing the Python runtime

There are 2 ways to access the Python runtime in your project code.

### Embedded C API.

You can use the [Python Embedded C
API](https://docs.python.org/3/extending/embedding.html) to instantiate a Python
interpreter. This is the approach taken by Briefcase; you may find the bootstrap
mainline code generated by Briefcase a helpful guide to what is needed to start
an interpreter and run Python code.

### PythonKit

An alternate approach is to use
[PythonKit](https://github.com/pvieito/PythonKit). PythonKit is a package that
provides a Swift API to running Python code.

To use PythonKit in your project:

1. Add PythonKit to your project using the Swift Package manager. See the
   PythonKit documentation for details.

2. Create a file called `module.modulemap` inside
   `Python.xcframework/macos-arm64_x86_64/Headers/`, containing the following
   code:
```
module Python {
    umbrella header "Python.h"
    export *
    link "Python"
}
```

3. In your Swift code, initialize the Python runtime. This should generally be
   done as early as possible in the application's lifecycle, but definitely
   needs to be done before you invoke Python code:
```swift
import Python

guard let stdLibPath = Bundle.main.path(forResource: "python-stdlib", ofType: nil) else { return }
guard let libDynloadPath = Bundle.main.path(forResource: "python-stdlib/lib-dynload", ofType: nil) else { return }
setenv("PYTHONHOME", stdLibPath, 1)
setenv("PYTHONPATH", "\(stdLibPath):\(libDynloadPath)", 1)
Py_Initialize()
// we now have a Python interpreter ready to be used
```

5. Invoke Python code in your app. For example:
```swift
import PythonKit

let sys = Python.import("sys")
print("Python Version: \(sys.version_info.major).\(sys.version_info.minor)")
print("Python Encoding: \(sys.getdefaultencoding().upper())")
print("Python Path: \(sys.path)")

_ = Python.import("math") // verifies `lib-dynload` is found and signed successfully
```

To integrate 3rd party python code and dependencies, you will need to make sure
`PYTHONPATH` contains their paths; once this has been done, you can run
`Python.import("<lib name>")`. to import that module from inside swift.
