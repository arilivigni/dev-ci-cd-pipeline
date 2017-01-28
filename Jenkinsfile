node('linch-pin-slave') {
  stage ('build') {
    dir('linch-pin') { // switch to subdir
        git url: 'http://git:8080/linch-pin.git'
        sh '''
            virtualenv $WORKSPACE/lp-test-venv
            source $WORKSPACE/lp-test-venv/bin/activate
            ./install.sh
        '''
    }
  }
  stage ('test') {
    sh '''
        source $WORKSPACE/lp-test-venv/bin/activate
        pip install nosexcover pylint pycrypto flake8 pep8
        iconv -f utf8 -t ascii $WORKSPACE/linch-pin/requirements.txt >> $WORKSPACE/linch-pin/requirements.txt
        chmod -x $WORKSPACE/linch-pin/tests/*.py
        nosetests --verbosity=3 --with-xunit $WORKSPACE/linch-pin/tests/test_linchpin_creds.py
    '''
    archiveArtifacts '*.xml,**/*.md'
    junit 'nosetests.xml'
  }
}
