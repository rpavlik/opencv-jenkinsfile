#!/usr/bin/env groovy

// Only keep the 10 most recent builds.
/*
properties([[$class: 'jenkins.model.BuildDiscarderProperty', strategy: [$class: 'LogRotator',
                                                                        numToKeepStr: '50',
                                                                        artifactNumToKeepStr: '20']]])
*/
def branch = (currentBuild.displayName.contains('3') ? 'master' : '2.4') // 2.4 or master

@NonCPS
def getTagPattern(br) {
    if (branch == '2.4') {
        return '[0-9].[0-9].[0-9].[0-9]'
    }
    return '[0-9].[0-9].[0-9]'
}

String tagname;
node('linux-build') {
    stage('Getting source and checking tag'){
        checkout changelog: false, poll: false, scm: [
            $class: 'GitSCM', branches: [[name: "*/${branch}"]],
            doGenerateSubmoduleConfigurations: false,
            extensions: [
                [$class: 'RelativeTargetDirectory', relativeTargetDir: "branch-${branch}"],
                [$class: 'CleanCheckout']
            ],
            submoduleCfg: [],
            userRemoteConfigs: [[url: 'https://github.com/itseez/OpenCV.git']]]

        // Get the most recent tag on this branch
        dir("branch-${branch}") {
            pattern = getTagPattern(branch)
            sh "git describe --abbrev=0 --match '${pattern}' > tagname"
            tagname = readFile('tagname').trim();
            echo "Latest ${branch} tag: ${tagname}"
            stash includes: 'tagname', name: 'tagname'
        }
    }

    stage("Getting the latest ${branch} tagged release"){
        echo "That tag name is ${tagname}"

        currentBuild.description = "Build of tag ${tagname}"
        checkout scm: [$class: 'GitSCM',
            branches: [[name: "refs/tags/${tagname}"]],
            doGenerateSubmoduleConfigurations: false,
            extensions: [
                [$class: 'CleanBeforeCheckout'],
                [$class: 'RelativeTargetDirectory', relativeTargetDir: 'latest-tag']
            ],
            submoduleCfg: [],
            userRemoteConfigs: [[url: 'https://github.com/itseez/OpenCV.git']] ]

        //checkout poll: false, scm: [$class: 'GitSCM', branches: [[name: "refs/tags/${tagname}"]], doGenerateSubmoduleConfigurations: false, extensions: [[$class: 'CleanBeforeCheckout'], [$class: 'RelativeTargetDirectory', relativeTargetDir: 'latest-tag']], submoduleCfg: [], userRemoteConfigs: [[url: 'https://github.com/itseez/OpenCV.git']]]
        //stash includes: 'latest-tag/**/*', name: 'sources'
    }
}

node('windows') {
    //stage 'Retrieving the stashed sources'
    //unstash 'sources'
    stage("Retrieving latest ${branch} tagged release on build node"){
        checkout scm: [$class: 'GitSCM',
            branches: [[name: "refs/tags/${tagname}"]],
            doGenerateSubmoduleConfigurations: false,
            extensions: [
                [$class: 'CleanBeforeCheckout'],
                [$class: 'RelativeTargetDirectory', relativeTargetDir: 'latest-tag']
                ],
            submoduleCfg: [],
            userRemoteConfigs: [[url: 'https://github.com/itseez/OpenCV.git']] ]

        windowsRmIfPresent 'install'
    }
    stage('Copying dependency artifacts'){

        windowsRmIfPresent 'deps'

        if (!fileExists('deps')) {
            bat 'mkdir deps'
        }
        dir('deps') {
            step([$class: 'CopyArtifact',
            projectName: 'Eigen-Vendored'
            ]);
        }
    }
    buildOpenCv '32', ['Release', 'Debug'], '14';
    buildOpenCv '64', ['Release', 'Debug'], '14';
    buildOpenCv '32', ['Release', 'Debug'], '15';
    buildOpenCv '64', ['Release', 'Debug'], '15';

    stage('Compressing and archiving results'){
        //unstash 'tagname'
        archive 'install/**/*'

        def sevenZipHome = tool name: '7zip', type: 'com.cloudbees.jenkins.plugins.customtools.CustomTool'
        def compressedFn = "opencv-${tagname}-build-${BUILD_ID}.7z"
        bat "${sevenZipHome}\\7za a -r ${compressedFn} install\\"
        archive compressedFn
    }
}

def windowsRmIfPresent(path) {
    if (fileExists(path)) {
        echo "Deleting ${path}"
        bat "del /s /f /q ${path}"
    }
}

def buildOpenCv(bits, configs, vsVer = '14') {
    def WORKSPACE = pwd();
    def generator = getVSGenerator(vsVer, bits)


    def eigen = "${WORKSPACE}/deps/install/include"
    def installPrefix = "${WORKSPACE}/install"
    def prefixPath = "${WORKSPACE}/deps/install"
    def buildDir = "build-${bits}"
    def srcDir = "${WORKSPACE}/latest-tag"

    stage("Configuring VS ${vsVer} ${bits}-bit build"){

        //windowsRmIfPresent buildDir
        configureOpenCvBuild(generator, srcDir, buildDir, installPrefix, prefixPath, eigen);
    }
    for (thisConfig in configs) {
        def config = thisConfig

        stage("Building and installing VS ${vsVer} ${bits}-bit ${config} build"){
            bat "cmake --build ${buildDir} --config ${config}"
            bat "cmake --build ${buildDir} --config ${config} --target INSTALL"
        }
    }
    windowsRmIfPresent "${buildDir}"
}

def configureOpenCvBuild(generator, srcDir, buildDir, installPrefix, prefixPath, eigenDir) {
    if (!fileExists(buildDir)) {
        bat "mkdir ${buildDir}"
    }
    dir(buildDir) {
        bat "cmake \"${srcDir}\" -G \"${generator}\" -DEIGEN3_INCLUDE_DIR=\"${eigenDir}\" -DCMAKE_INSTALL_PREFIX=${installPrefix} -DCMAKE_PREFIX_PATH=${prefixPath} " +
        "-DBUILD_PERF_TESTS=OFF " +
        "-DBUILD_opencv_apps=OFF " +
        "-DBUILD_opencv_contrib=OFF " +
        "-DBUILD_opencv_nonfree=OFF " +
        "-DBUILD_TESTS=OFF " +
        "-DBUILD_EXAMPLES=OFF " +
        "-DWITH_EIGEN=ON "
    }
}

/* ComputeGenerator.groovy by Ryan Pavlik - maintained at https://gist.github.com/rpavlik/3ad0e691e5d51606bd67 */

/* Uncomment the following for testing purposes only */
/*
VS='12'
BIT='64'
*/
/* should give result [GENERATOR:Visual Studio 12 2013 Win64] */

@NonCPS
def getVSVersionName(vsVerNum) {
    def result = 'Visual Studio ' + vsVerNum
    switch (vsVerNum) {
        case '8':
        result += ' 2005'
        break
        case '9':
        result += ' 2008'
        break
        case '10':
        result += ' 2010'
        break
        case '11':
        result += ' 2012'
        break
        case '12':
        result += ' 2013'
        break
        case '14':
        result += ' 2015'
        break
        case '15':
        result += ' 2017'
        break
    }
    result
}
@NonCPS
def getVSGenerator(vsVerNum, bits) {
    def result = getVSVersionName(vsVerNum)
    if (bits.equals('64')) {
        result += ' Win64';
    }
    result
}
