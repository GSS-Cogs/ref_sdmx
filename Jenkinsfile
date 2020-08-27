pipeline {
    agent {
        label 'master'
    }
    stages {
        stage('Clean') {
            steps {
                sh 'rm -rf out'
            }
        }
        stage('Fetch') {
            steps {
                script {
                    httpRequest(httpMode: 'GET',
                                url: 'https://registry.sdmx.org/ws/public/sdmxapi/rest/codelist/ESTAT/CL_UNIT/1.2',
                                outputFile: 'sdmx-cl_unit.xml')
                }
            }
        }
        stage('Transform') {
            agent {
                dockerfile {
                    filename 'Dockerfile'
                    args "-v ${env.WORKSPACE}:/workspace"
                    reuseNode true
                }
            }
            steps {
                script {
                    sh "xsltproc -o sdmx-cl_unit.rdf sdmx2skos.xslt sdmx-cl_unit.xml"
                }
            }
        }
        stage('Add concept scheme') {
            steps {
                script {
                    def pmd = pmdConfig('pmd')
                    for (myDraft in pmd.drafter
                            .listDraftsets(Drafter.Include.OWNED)
                            .findAll { it['display-name'] == env.JOB_NAME }) {
                        pmd.drafter.deleteDraftset(myDraft.id)
                    }
                    def id = pmd.drafter.createDraftset(env.JOB_NAME).id
                    for (graph in util.jobGraphs(pmd, id)) {
                        pmd.drafter.deleteGraph(id, graph)
                        echo "Removing own graph ${graph}"
                    }
                    for (def rdf : findFiles(glob: "*.rdf")) {
                        String graph = sh (
                                script: '''sed -n -e 's/.*xml:base="\\([^ ]*\\)".*/\\1/p' sdmx-cl_unit.rdf''',
                                returnStdout: true
                        ).trim()
                        pmd.drafter.addData(id, "${WORKSPACE}/${rdf.name}", "application/rdf+xml", "UTF-8", graph)
                        writeFile(file: "${rdf.name}-prov.ttl}", text: util.jobPROV(graph))
                        pmd.drafter.addData(id, "${WORKSPACE}/${rdf.name}-prov.ttl", "text/turtle", "UTF-8", graph)
                    }
                    pmd.drafter.publishDraftset(id)
                }
            }
        }
    }
}
